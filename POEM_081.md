POEM ID: 081  
Title: Add Subproblem Component to standard component set.  
authors: nsteffen (Nate Steffen)  
Competing POEMs:  
Related POEMs:  
Associated implementation PR: #2817  

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
"""Define the SubproblemComp class for evaluating OpenMDAO systems within problems."""

from openmdao.core.explicitcomponent import ExplicitComponent
from openmdao.core.problem import Problem
from openmdao.core.constants import _UNDEFINED
from openmdao.utils.general_utils import find_matches
from openmdao.utils.om_warnings import issue_warning
from openmdao.core.driver import Driver


def _get_model_vars(varType, vars, model_vars):
    """
    Get the requested IO variable data from model's list of IO.

    Parameters
    ----------
    varType : str
        Specifies whether inputs or outputs are being extracted.
    vars : list of str or tuple
        List of provided var names in str or tuple form. If an element is a str,
        then it should be the absolute name or the promoted name in its group. If it is a tuple,
        then the first element should be the absolute name or group's promoted name, and the
        second element should be the var name you wish to refer to it within the subproblem
        [e.g. (path.to.var, desired_name)].
    model_vars : list of tuples
        List of model variable absolute names and meta data.

    Returns
    -------
    dict
        Dict to update `self.options` with desired IO data in `SubproblemComp`.
    """
    var_dict = {varType: {}}

    # check for wildcards and append them to vars list
    wildcards = [i for i in vars if isinstance(i, str) and '*' in i]
    for i in wildcards:
        vars.extend(find_matches(i, [meta['prom_name'] for _, meta in model_vars]))
        vars.remove(i)

    for var in vars:
        if isinstance(var, tuple):
            # check if user tries to use wildcard in tuple
            if '*' in var[0] or '*' in var[1]:
                raise Exception('Cannot use \'*\' in tuple variable.')

            # check if variable already exists in var_dict[varType]
            # i.e. no repeated variable names
            if var[1] in var_dict[varType]:
                raise Exception(f'Variable {var[1]} already exists. Rename variable'
                                ' or delete copy of variable.')

            # make dict with given var name as key and meta data from model_vars
            # check if name == var[0] -> var[0] is abs name and var[1] is alias
            # check if meta['prom_name'] == var[0] -> var[0] is prom name and var[1] is alias
            tmp_dict = {var[1]: meta for name, meta in model_vars
                        if name == var[0] or meta['prom_name'] == var[0]}

            # check if dict is empty (no vars added)
            if len(tmp_dict) == 0:
                raise Exception(f'Promoted name {var[0]} does not'
                                ' exist in model.')

            var_dict[varType].update(tmp_dict)

        elif isinstance(var, str):
            # check if variable already exists in var_dict[varType]
            # i.e. no repeated variable names
            if var in var_dict[varType]:
                raise Exception(f'Variable {var} already exists. Rename variable'
                                ' or delete copy of variable.')

            # make dict with given var name as key and meta data from model_vars
            # check if name == var -> given var is abs name
            # check if meta['prom_name'] == var -> given var is prom_name
            # check if name.endswith('.' + var) -> given var is last part of abs name
            tmp_dict = {var: meta for name, meta in model_vars
                        if name == var or meta['prom_name'] == var}

            # check if provided variable appears more than once in model
            if len(tmp_dict) > 1:
                raise Exception(f'Ambiguous variable {var}. To'
                                ' specify which one is desired, use a tuple'
                                ' with the promoted name and variable name'
                                ' instead [e.g. (prom_name, var_name)].')

            # checks if provided variable doesn't exist in model
            elif len(tmp_dict) == 0:
                raise Exception(f'Variable {var} does not exist in model.')

            var_dict[varType].update(tmp_dict)

        else:
            raise Exception(f'Type {type(var)} is invalid. Must be'
                            ' string or tuple.')

    return var_dict


class SubproblemComp(ExplicitComponent):
    """
    System level container for systems and drivers.

    Parameters
    ----------
    model : <System>
        The system-level <System>.
    inputs : list of str or tuple
        List of provided input names in str or tuple form. If an element is a str,
        then it should be the absolute name or the promoted name in its group. If it is a tuple,
        then the first element should be the absolute name or group's promoted name, and the
        second element should be the var name you wish to refer to it within the subproblem
        [e.g. (path.to.var, desired_name)].
    outputs : list of str or tuple
        List of provided output names in str or tuple form. If an element is a str,
        then it should be the absolute name or the promoted name in its group. If it is a tuple,
        then the first element should be the absolute name or group's promoted name, and the
        second element should be the var name you wish to refer to it within the subproblem
        [e.g. (path.to.var, desired_name)].
    driver : <Driver> or None
        The driver for the problem. If not specified, a simple "Run Once" driver will be used.
    comm : MPI.Comm or <FakeComm> or None
        The global communicator.
    name : str or None
        Problem name. Can be used to specify a Problem instance when multiple Problems
        exist.
    reports : str, bool, None, _UNDEFINED
        If _UNDEFINED, the OPENMDAO_REPORTS variable is used. Defaults to _UNDEFINED.
        If given, reports overrides OPENMDAO_REPORTS. If boolean, enable/disable all reports.
        Since none is acceptable in the environment variable, a value of reports=None
        is equivalent to reports=False. Otherwise, reports may be a sequence of
        strings giving the names of the reports to run.
    prob_options : dict or None
        Remaining named args for problem that are converted to options.
    **kwargs : named args
        All remaining named args that become options for `SubproblemComp`.

    Attributes
    ----------
    prob_args : dict
        Extra arguments to be passed to the problem instantiation.
    model : <System>
        The system being analyzed in subproblem.
    model_input_names : list of str or tuple
        List of inputs requested by user to be used as inputs in the
        subproblem's system.
    model_output_names : list of str or tuple
        List of outputs requested by user to be used as outputs in the
        subproblem's system.
    """

    def __init__(self, model, inputs, outputs, driver=None, comm=None,
                 name=None, reports=_UNDEFINED, prob_options=None, **kwargs):
        """
        Initialize all attributes.
        """
        # check for driver and issue warning about its current use
        # in subproblem
        if driver is not None:
            issue_warning('Driver results may not be accurate if'
                          ' derivatives are needed. Set driver to'
                          ' None if your subproblem isn\'t reliant on'
                          ' a driver.')

        # make `prob_options` empty dict to be passed as **options to problem
        # instantiation
        if prob_options is None:
            prob_options = {}

        # call base class to set kwargs
        super().__init__(**kwargs)

        # store inputs and outputs in options
        self.options.declare('inputs', {}, types=dict,
                             desc='Subproblem Component inputs')
        self.options.declare('outputs', {}, types=dict,
                             desc='Subproblem Component outputs')

        # set other variables necessary for subproblem

        self.prob_args = {'driver': driver,
                          'comm': comm,
                          'name': name,
                          'reports': reports}

        self.prob_args.update(prob_options)

        self.model = model
        self.model_input_names = inputs
        self.model_output_names = outputs

    def setup(self):
        """
        Perform some final setup and checks.
        """
        p = self._subprob = Problem(**self.prob_args)
        p.model.add_subsystem('subsys', self.model, promotes=['*'])

        p.setup(force_alloc_complex=self._problem_meta['force_alloc_complex'])
        p.final_setup()

        model_inputs = p.model.list_inputs(out_stream=None, prom_name=True,
                                           units=True, shape=True, desc=True)
        model_outputs = p.model.list_outputs(out_stream=None, prom_name=True,
                                             units=True, shape=True, desc=True)

        self.options.update(_get_model_vars('inputs', self.model_input_names, model_inputs))
        self.options.update(_get_model_vars('outputs', self.model_output_names, model_outputs))

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
            meta['prom_name'] = prom_name
            self._input_names.append(var)

        for var, meta in outputs.items():
            prom_name = meta.pop('prom_name')
            self.add_output(var, **meta)
            meta['prom_name'] = prom_name
            self._output_names.append(var)

            for ip in self._input_names:
                self.declare_partials(of=var, wrt=ip)

    def _set_complex_step_mode(self, active):
        super()._set_complex_step_mode(active)
        self._subprob.set_complex_step_mode(active)

    def compute(self, inputs, outputs):
        """
        Perform the subproblem system computation at run time.

        Parameters
        ----------
        inputs : Vector
            Unscaled, dimensional input variables read via inputs[key].
        outputs : Vector
            Unscaled, dimensional output variables read via outputs[key].
        """
        p = self._subprob

        # switch subproblem to use complex IO if in complex step mode
        self._set_complex_step_mode(self.under_complex_step)

        # setup input values
        for inp in self._input_names:
            p.set_val(self.options['inputs'][inp]['prom_name'], inputs[inp])

        if not isinstance(p.driver, Driver):
            p.run_driver()
        else:
            p.run_model()

        # store output vars
        for op in self._output_names:
            outputs[op] = p.get_val(self.options['outputs'][op]['prom_name'])

    def compute_partials(self, inputs, partials):
        """
        Collect computed partial derivatives and return them.

        Checks if the needed derivatives are cached already based on the
        inputs vector. Refreshes the cache by re-computing the current point
        if necessary.

        Parameters
        ----------
        inputs : Vector
            Unscaled, dimensional input variables read via inputs[key].
        partials : Jacobian
            Sub-jac components written to partials[output_name, input_name].
        """
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
from openmdao.core.SubproblemComp import SubproblemComp


prob = om.Problem()

model = om.ExecComp('z = x**2 + y')
submodel1 = om.ExecComp('x = r*cos(theta)')
submodel2 = om.ExecComp('y = r*sin(theta)')

subprob1 = SubproblemComp(model=submodel1, inputs=['r', 'theta'],
                          outputs=['x'])
subprob2 = SubproblemComp(model=submodel2, inputs=['r', 'theta'],
                          outputs=['y'])

prob.model.add_subsystem('sub1', subprob1, promotes_inputs=['r','theta'],
                            promotes_outputs=['x'])
prob.model.add_subsystem('sub2', subprob2, promotes_inputs=['r','theta'],
                            promotes_outputs=['y'])
prob.model.add_subsystem('supModel', model, promotes_inputs=['x','y'],
                            promotes_outputs=['z'])

prob.setup(force_alloc_complex=True)

prob.set_val('r', 1)
prob.set_val('theta', pi)

prob.run_model()
cpd = prob.check_partials(method='cs')    
print(f"x = {prob.get_val('x')}")
print(f"y = {prob.get_val('y')}") 
print(f"z = {prob.get_val('z')}")

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
x = [-1.]
y = [1.2246468e-16]
z = [1.]
```
