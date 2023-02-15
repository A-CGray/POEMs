POEM ID: 081  
Title: Add Subproblem Component to standard component set.  
authors: @nsteffen  
Competing POEMs:  
Related POEMs:  
Associated implementation PR: #TBD  

Status:

- [x] Active
- [ ] Requesting decision
- [ ] Accepted 
- [ ] Rejected
- [ ] Integrated

## Motivation

Sometimes, the user might wish to set up a problem that has models that evaluate OpenMDAO systems themselves. However, there is currently no simple way to do this within OpenMDAO.

As the depth of a system increases, the input and output vectors can grow quite large. This increases the complexity of the interface and can slow its performance, and OpenMDAO doesn't have any mechanism to remedy this.

The current OpenMDAO API doesn't have an intuitive way of nesting drivers (wrapping a driver around a problem with a driver).

## Proposed Solution

A new subproblem component will allow for a user to modularize their OpenMDAO systems. Not only can this "hide" certain inputs from the top-level model and reduce its input vector size, but it allows for OpenMDAO systems to be evaluated within an overall OpenMDAO system.

Furthermore, it has the added benefit of allowing for nested drivers. The OpenMDAO system within a subproblem can be subject to constraints, have design variables, and objectives at its system level. This makes implementing nested drivers more intuitive for users.

The code below is how the new subproblem component will look. Its inputs are similar to that of `Problem`, and since it is an `om.ExplicitComponent`, it can be added as a subsystem to a model in a top-level problem.

## Prototype Implementation

```python
import openmdao.api as om
from openmdao.core.constants import _UNDEFINED
from openmdao.utils.om_warnings import issue_warning
from openmdao.core.driver import Driver


class SubproblemComp(om.ExplicitComponent):
    """
    System level container for systems and drivers.
    
    Parameters
    ----------
    problem : Problem
        Problem instance that becomes the subproblem.
    model : <System> or None
        The system-level <System>. Required for subproblem.
    inputs : list of str or tuple
        List of desired inputs to subproblem. If an element is a str, then it should be
        the var name in its promoted name. If it is a tuple, then the first element 
        should be the promoted name, and the second element should be the var name
        you wish to refer to it by within the subproblem [e.g. (prom_name, var_name)].
    outputs : list of str or tuple
        List of desired outputs from subproblem. If an element is a str, then it should be
        the var name in its promoted name. If it is a tuple, then the first element 
        should be the promoted name, and the second element should be the var name
        you wish to refer to it by within the subproblem [e.g. (prom_name, var_name)].
    **kwargs : named args
        All remaining named args that become options for `SubproblemComp`.
    """
    def __init__(self, problem=None, model=None, inputs=None,
                 outputs=None, **kwargs):

        # check the type because two `Driver` instances are not equal
        # om.ScipyOptimizeDriver and om.pyOptSparseDriver
        if type(problem.driver) != type(Driver()):
            issue_warning('Driver results may not be accurate if'
                          ' derivatives are needed. Set driver to'
                          ' None if your subproblem isn\'t reliant on'
                          ' a driver.')

        # call base class to set kwargs
        super().__init__(**kwargs)

        # store inputs and outputs in options
        self.options.declare('inputs', {}, types=dict,
                             desc='Subproblem Component inputs')
        self.options.declare('outputs', {}, types=dict,
                             desc='Subproblem Component outputs')

        # set other variables necessary for subproblem
        self._prev_complex_step = False
        
        p = self._subprob = problem
        p.model.add_subsystem('subsys', model, promotes=['*'])

        p.setup(force_alloc_complex=False)
        p.final_setup()

        model_inputs = p.model.list_inputs(out_stream=None, prom_name=True,
                                           units=True, shape=True, desc=True)
        model_outputs = p.model.list_outputs(out_stream=None, prom_name=True,
                                             units=True, shape=True, desc=True)

        # store model inputs/outputs as dictionary with keys as the promoted name
        model_inputs = {meta['prom_name']: meta for _, meta in model_inputs}
        model_outputs = {meta['prom_name']: meta for _, meta in model_outputs}

        # loop through inputs and make sure they're valid for use
        for inp in inputs:
            if type(inp) is tuple:
                # check if variable already exists in self.options['inputs']
                # i.e. no repeated input variable names
                inp_vars = list(self.options['inputs'].keys())
                if inp[1] in inp_vars:
                    raise Exception(f'Variable {inp[1]} already exists. Rename variable'
                                    ' or delete copy of variable.')

                # make dict with given var name as key and meta data from model_inputs
                inp_dict = {inp[1]: meta for _,meta in model_inputs.items()
                            if meta['prom_name'] == inp[0]}

                # check if dict is empty (no inputs added)
                if len(inp_dict) == 0:
                    raise Exception(f'Promoted name {inp[0]} does not'
                                    ' exist in model.')

                # update options inputs dict with new input dict
                self.options['inputs'].update(inp_dict)

            elif type(inp) is str:
                # check if variable already exists in self.options['inputs']
                # i.e. no repeated input variable names
                inp_vars = list(self.options['inputs'].keys())
                if inp in inp_vars:
                    raise Exception(f'Variable {inp} already exists. Rename variable'
                                    ' or delete copy of variable.')

                # make dict with given var name as key and meta data from model_inputs
                inp_dict = {inp: meta for _,meta in model_inputs.items()
                            if meta['prom_name'].endswith(inp)}

                # check if provided variable appears more than once in model
                if len(inp_dict) > 1:
                    raise Exception(f'Ambiguous variable {inp} in inputs. To'
                                    ' specify which one is desired, use a tuple'
                                    ' with the promoted name and variable name'
                                    ' instead [e.g. (prom_name, var_name)].')

                # checks if provided variable doesn't exist in model
                elif len(inp_dict) == 0:
                    raise Exception(f'Variable {inp} does not exist in model.')

                # update options inputs dict with new input dict
                self.options['inputs'].update(inp_dict)

            else:
                raise Exception(f'Type {type(inp)} is invalid for input. Must be'
                                ' string or tuple.')

        # loop through outputs and make sure they're valid for use
        for out in outputs:
            if type(out) is tuple:
                # check if variable already exists in self.options['outputs']
                # i.e. no repeated output variable names
                out_vars = list(self.options['outputs'].keys())
                if out[1] in out_vars:
                    raise Exception(f'Variable {out[1]} already exists. Rename variable'
                                    ' or delete copy of variable.')

                # make dict with given var name as key and meta data from model_outputs
                out_dict = {out[1]: meta for _,meta in model_outputs.items()
                            if meta['prom_name'] == out[0]}

                # checks if provided variable doesn't exist in model
                if len(out_dict) == 0:
                    raise Exception(f'Variable {out[0]} does not exist in model.')

                # update options outputs dict with new output dict
                self.options['outputs'].update(out_dict)

            elif type(out) is str:
                # check if variable already exists in self.options['outputs']
                # i.e. no repeated output variable names
                out_vars = list(self.options['outputs'].keys())
                if out in out_vars:
                    raise Exception(f'Variable {out} already exists. Rename variable'
                                    ' or delete copy of variable.')

                # make dict with given var name as key and meta data from model_outputs
                out_dict = {out: meta for _,meta in model_outputs.items()
                            if meta['prom_name'].endswith(out)}

                # check if provided variable appears more than once in model
                if len(out_dict) > 1:
                    raise Exception(f'Ambiguous variable {out} in outputs. To'
                                    ' specify which one is desired, use a tuple'
                                    ' with the promoted name and variable name'
                                    ' instead [e.g. (prom_name, var_name)].')

                 # checks if provided variable doesn't exist in model
                elif len(out_dict) == 0:
                    raise Exception(f'Variable {out} does not exist in model.')

                # update options outputs dict with new output dict
                self.options['outputs'].update(out_dict)

            else:
                raise Exception(f'Type {type(out)} is invalid for output. Must be'
                                ' string or tuple.')

    def setup(self):
        inputs = self.options['inputs']
        outputs = self.options['outputs']

        # instantiate input/output name list for use in compute and 
        # compute partials
        self._input_names = []
        self._output_names = []

        # remove the `prom_name` from the metadata and then store it for each
        # input and output
        for var, meta in inputs.items():
            prom_name = meta.pop('prom_name')
            self.add_input(var, **meta)
            # add prom_name back
            meta['prom_name'] = prom_name
            self._input_names.append(var)

        for var, meta in outputs.items():
            prom_name = meta.pop('prom_name')
            self.add_output(var, **meta)
            # add prom_name back
            meta['prom_name'] = prom_name
            self._output_names.append(var)

            for ip in self._input_names:
                self.declare_partials(of=var, wrt=ip)
    
    def compute(self, inputs, outputs):
        p = self._subprob

        # switch subproblem to use complex IO if in complex step mode
        if self.under_complex_step != self._prev_complex_step:
            if self.under_complex_step:
                p.setup(force_alloc_complex=True)
                p.final_setup()
                p.set_complex_step_mode(True)
            else:
                p.setup()
                p.final_setup()
                p.set_complex_step_mode(False)
            self._prev_complex_step = self.under_complex_step

        # setup input values
        for inp in self._input_names:
            p.set_val(self.options['inputs'][inp]['prom_name'], inputs[inp])

        # use driver if it is provided
        if type(p.driver) != type(Driver()):
            p.run_driver()
        else:
            p.run_model()

        # store output vars
        for op in self._output_names:
            outputs[op] = p.get_val(self.options['outputs'][op]['prom_name'])

    def compute_partials(self, inputs, partials):
        p = self._subprob
        for inp in self._input_names:
            p.set_val(self.options['inputs'][inp]['prom_name'], inputs[inp])

        # compute total derivatives for now... assuming every output is sensitive 
        # to every input. Will be changed in a future version
        tots = p.compute_totals(of=self._output_names, wrt=self._input_names,
                                use_abs_names=False)

        # store derivatives in Jacobian
        for of in self._output_names:
            for wrt in self._input_names:
                partials[of, wrt] = tots[of, wrt]
```

## Example

```python
from numpy import pi
import openmdao.api as om
from SubproblemComp import SubproblemComp


prob = om.Problem()

model = om.ExecComp('z = x**2 + y')
sub_model1 = om.ExecComp('x = r*cos(theta)')
sub_model2 = om.ExecComp('y = r*sin(theta)')

subprob1 = SubproblemComp(problem=om.Problem(), model=sub_model1,
                          inputs=['r', 'theta'], outputs=['x'])
subprob2 = SubproblemComp(problem=om.Problem(), model=sub_model2,
                          inputs=['r', 'theta'], outputs=['y'])

prob.model.add_subsystem('supModel', model, promotes_inputs=['x','y'],
                            promotes_outputs=['z'])
prob.model.add_subsystem('sub1', subprob1, promotes_inputs=['r','theta'],
                            promotes_outputs=['x'])
prob.model.add_subsystem('sub2', subprob2, promotes_inputs=['r','theta'],
                            promotes_outputs=['y'])

prob.setup(force_alloc_complex=True)

prob.set_val('r', 1)
prob.set_val('theta', pi)

prob.run_model()
cpd = prob.check_partials(method='cs')     
print(prob.get_val('z'))

# om.n2(prob)
```

## Outputs

```
--------------------------------
Component: SubproblemComp 'sub1'
--------------------------------

  sub1: 'x' wrt 'r'
    Analytic Magnitude: 1.000000e+00
          Fd Magnitude: 1.000000e+00 (cs:None)
    Absolute Error (Jan - Jfd) : 0.000000e+00

    Relative Error (Jan - Jfd) / Jfd : 0.000000e+00

    Raw Analytic Derivative (Jfor)
[[-1.]]

    Raw CS Derivative (Jcs)
[[-1.]]
 - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
  sub1: 'x' wrt 'theta'
    Analytic Magnitude: 1.224647e-16
          Fd Magnitude: 1.224647e-16 (cs:None)
    Absolute Error (Jan - Jfd) : 0.000000e+00

    Relative Error (Jan - Jfd) / Jfd : 0.000000e+00

    Raw Analytic Derivative (Jfor)
[[-1.2246468e-16]]

    Raw CS Derivative (Jcs)
[[-1.2246468e-16]]
 - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
--------------------------------
Component: SubproblemComp 'sub2'
--------------------------------

  sub2: 'y' wrt 'r'
    Analytic Magnitude: 1.224647e-16
          Fd Magnitude: 1.224647e-16 (cs:None)
    Absolute Error (Jan - Jfd) : 0.000000e+00

    Relative Error (Jan - Jfd) / Jfd : 0.000000e+00

    Raw Analytic Derivative (Jfor)
[[1.2246468e-16]]

    Raw CS Derivative (Jcs)
[[1.2246468e-16]]
 - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
  sub2: 'y' wrt 'theta'
    Analytic Magnitude: 1.000000e+00
          Fd Magnitude: 1.000000e+00 (cs:None)
    Absolute Error (Jan - Jfd) : 0.000000e+00

    Relative Error (Jan - Jfd) / Jfd : 0.000000e+00

    Raw Analytic Derivative (Jfor)
[[-1.]]

    Raw CS Derivative (Jcs)
[[-1.]]
 - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
[2.]
```
