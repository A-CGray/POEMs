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
from openmdao.core.constants import _SetupStatus


def _get_model_vars(vars, model_vars):
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
    var_dict = {}

    # check for wildcards and append them to vars list
    patterns = [i for i in vars if isinstance(i, str)]
    var_list = [meta['prom_name'] for _, meta in model_vars]
    for i in patterns:
        matches = find_matches(i, var_list)
        if len(matches) == 0:
            continue
        vars.extend(matches)
        vars.remove(i)

    for var in vars:
        if isinstance(var, tuple):
            # check if user tries to use wildcard in tuple
            if '*' in var[0] or '*' in var[1]:
                raise Exception('Cannot use \'*\' in tuple variable.')

            # check if variable already exists in var_dict[varType]
            # i.e. no repeated variable names
            if var[1] in var_dict:
                raise Exception(f'Variable {var[1]} already exists. Rename variable'
                                ' or delete copy.')

            # make dict with given var name as key and meta data from model_vars
            # check if name == var[0] -> var[0] is abs name and var[1] is alias
            # check if meta['prom_name'] == var[0] -> var[0] is prom name and var[1] is alias
            tmp_dict = {var[1]: meta for name, meta in model_vars
                        if name == var[0] or meta['prom_name'] == var[0]}

            # check if dict is empty (no vars added)
            if len(tmp_dict) == 0:
                raise Exception(f'Promoted name {var[0]} does not'
                                ' exist in model.')

            var_dict.update(tmp_dict)

        elif isinstance(var, str):
            # check if variable already exists in var_dict[varType]
            # i.e. no repeated variable names
            if var in var_dict:
                raise Exception(f'Variable {var} already exists. Rename variable'
                                ' or delete copy.')

            # make dict with given var name as key and meta data from model_vars
            # check if name == var -> given var is abs name
            # check if meta['prom_name'] == var -> given var is prom_name
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

            var_dict.update(tmp_dict)

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
    inputs : list of str or tuple or None
        List of provided input names in str or tuple form. If an element is a str,
        then it should be the absolute name or the promoted name in its group. If it is a tuple,
        then the first element should be the absolute name or group's promoted name, and the
        second element should be the var name you wish to refer to it within the subproblem
        [e.g. (path.to.var, desired_name)].
        If inputs is None, then inputs not connected to outputs from driver design variables are
        used.
    outputs : list of str or tuple or None
        List of provided output names in str or tuple form. If an element is a str,
        then it should be the absolute name or the promoted name in its group. If it is a tuple,
        then the first element should be the absolute name or group's promoted name, and the
        second element should be the var name you wish to refer to it within the subproblem
        [e.g. (path.to.var, desired_name)].
        If outputs is None, then outputs not connected to outputs that are driver design variables
        and are not tagged as `openmdao:indep_var` are used.
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
    is_set_up : bool
        Flag to determne if subproblem is set up. Used for add_input/add_output to
        determine how to add the io.
    _input_names : list of str
        List of inputs added to model.
    _output_names : list of str
        List of outputs added to model.
    subprob_design_vars : dict
        Dict of design vars and their meta data.
    subprob_constraints : dict
        Dict of constraints and their meta data.
    subprob_objectives : dict
        Dict of objectives and their meta data.
    """

    def __init__(self, model, inputs=None, outputs=None, driver=None,
                 comm=None, name=None, reports=_UNDEFINED, prob_options=None, **kwargs):
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
        self.model_input_names = inputs if inputs is not None else []
        self.model_output_names = outputs if outputs is not None else []
        self.is_set_up = False
        self.subprob_design_vars = {}
        self.subprob_constraints = {}
        self.subprob_objectives = {}

    def add_input(self, name):
        """
        Add input to model before or after setup.

        Parameters
        ----------
        name : str
            Name of input to be added.
        """
        if not self.is_set_up:
            self.model_input_names.append(name)
            return

        if self._problem_meta['setup_status'] > _SetupStatus.POST_CONFIGURE:
            raise Exception('Cannot call add_input after configure.')

        self.options['inputs'].update(_get_model_vars([name], self.boundary_inputs))

        meta = self.options['inputs'][name]

        prom_name = meta.pop('prom_name')
        super().add_input(name, **meta)
        meta['prom_name'] = prom_name
        self._input_names.append(name)

    def add_output(self, name):
        """
        Add output to model before or after setup.

        Parameters
        ----------
        name : str
            Name of output to be added.
        """
        if not self.is_set_up:
            self.model_output_names.append(name)
            return

        if self._problem_meta['setup_status'] > _SetupStatus.POST_CONFIGURE:
            raise Exception('Cannot call add_output after configure.')

        self.options['outputs'].update(_get_model_vars([name], self.all_outputs))

        meta = self.options['outputs'][name]

        prom_name = meta.pop('prom_name')
        super().add_output(name, **meta)
        meta['prom_name'] = prom_name
        self._output_names.append(name)

    def add_design_var(self, name, **kwargs):
        """
        Add design variable before or after setup.

        Parameters
        ----------
        name : str
            Name of design variable to be added.
        **kwargs : named args
            Meta data associated with design variables.
        """
        if not self.is_set_up:
            self.subprob_design_vars.update({name: kwargs})
            return

        if self._problem_meta['setup_status'] > _SetupStatus.POST_CONFIGURE:
            raise Exception('Cannot call add_output after configure.')

        self._subprob.model.add_design_var(self.options['inputs'][name]['prom_name'], **kwargs)

    def add_constraint(self, name, **kwargs):
        """
        Add constraint before or after setup.

        Parameters
        ----------
        name : str
            Name of constraint to be added.
        **kwargs : named args
            Meta data associated with constraint.
        """
        if not self.is_set_up:
            self.subprob_constraints.update({name: kwargs})
            return

        if self._problem_meta['setup_status'] > _SetupStatus.POST_CONFIGURE:
            raise Exception('Cannot call add_output after configure.')

        self._subprob.model.add_constraint(self.options['outputs'][name]['prom_name'], **kwargs)

    def add_objective(self, name, **kwargs):
        """
        Add objective before or after setup.

        Parameters
        ----------
        name : str
            Name of objectiveto be added.
        **kwargs : named args
            Meta data associated with objective.
        """
        if not self.is_set_up:
            self.subprob_objectives.update({name: kwargs})
            return

        if self._problem_meta['setup_status'] > _SetupStatus.POST_CONFIGURE:
            raise Exception('Cannot call add_output after configure.')

        self._subprob.model.add_objective(self.options['outputs'][name]['prom_name'], **kwargs)

    def setup(self):
        """
        Perform some final setup and checks.
        """
        if self.is_set_up is False:
            self.is_set_up = True

        p = self._subprob = Problem(**self.prob_args)
        p.model.add_subsystem('subsys', self.model, promotes=['*'])

        # perform first setup to be able to get inputs and outputs
        # if subprob.setup is called before the top problem setup, we can't rely
        # on using the problem meta data, so default to False
        if self._problem_meta is None:
            p.setup(force_alloc_complex=False)
        else:
            p.setup(force_alloc_complex=self._problem_meta['force_alloc_complex'])
        p.final_setup()

        # boundary inputs are any inputs that externally come into `SubproblemComp`
        self.boundary_inputs = p.model.list_inputs(out_stream=None, prom_name=True,
                                                   units=True, shape=True, desc=True,
                                                   is_indep_var=True)
        # want all outputs from the `SubproblemComp`, including ivcs/design vars
        self.all_outputs = p.model.list_outputs(out_stream=None, prom_name=True,
                                                units=True, shape=True, desc=True)

        self.options['inputs'].update(_get_model_vars(self.model_input_names, self.boundary_inputs))
        self.options['outputs'].update(_get_model_vars(self.model_output_names, self.all_outputs))

        inputs = self.options['inputs']
        outputs = self.options['outputs']

        self._input_names = []
        self._output_names = []

        for var, meta in inputs.items():
            prom_name = meta.pop('prom_name')
            super().add_input(var, **meta)
            meta['prom_name'] = prom_name
            self._input_names.append(var)

        for var, meta in outputs.items():
            prom_name = meta.pop('prom_name')
            super().add_output(var, **meta)
            meta['prom_name'] = prom_name
            self._output_names.append(var)

    def _setup_var_data(self):
        super()._setup_var_data()

        p = self._subprob
        inputs = self.options['inputs']
        outputs = self.options['outputs']

        # can't have coloring if there are no io declared
        if len(inputs) == 0 or len(outputs) == 0:
            return

        for name, meta in self.subprob_design_vars.items():
            p.model.add_design_var(inputs[name]['prom_name'], **meta)

        for name, meta in self.subprob_constraints.items():
            p.model.add_constraint(outputs[name]['prom_name'], **meta)

        for name, meta in self.subprob_objectives.items():
            p.model.add_objective(outputs[name]['prom_name'], **meta)

        p.driver.declare_coloring()

        # setup again to compute coloring
        if self._problem_meta is None:
            p.setup(force_alloc_complex=False)
        else:
            p.setup(force_alloc_complex=self._problem_meta['force_alloc_complex'])
        p.final_setup()

        # get coloring and change row and column names to be prom names for use later
        self.coloring = p.driver._get_coloring(run_model=True)
        if self.coloring is not None:
            self.coloring._row_vars = [meta['prom_name'] for name, meta in self.all_outputs
                                       if name in self.coloring._row_vars]
            self.coloring._col_vars = [meta['prom_name'] for _, meta in self.boundary_inputs]

        if self.coloring is None:
            self.declare_partials(of='*', wrt='*')
        else:
            for of, wrt, nzrows, nzcols, _, _, _, _ in self.coloring._subjac_sparsity_iter():
                self.declare_partials(of=of, wrt=wrt, rows=nzrows, cols=nzcols)

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

        for inp in self._input_names:
            p.set_val(self.options['inputs'][inp]['prom_name'], inputs[inp])

        p.driver.run()

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

        tots = p.driver._compute_totals(of=self._output_names, wrt=self._input_names,
                                        use_abs_names=False)

        if self.coloring is None:
            for key, tot in tots.items():
                partials[key] = tot
        else:
            for of, wrt, nzrows, nzcols, _, _, _, _ in self.coloring._subjac_sparsity_iter():
                partials[of, wrt] = tots[of, wrt][nzrows, nzcols].ravel()


```

## Example

```python
from numpy import pi
import openmdao.api as om


p = om.Problem()

model = om.Group()
model.add_subsystem('supComp', om.ExecComp('z = x**2 + y'),
                    promotes_inputs=['x', 'y'],
                    promotes_outputs=['z'])

submodel1 = om.Group()
submodel1.add_subsystem('sub1_ivc_r', om.IndepVarComp('r', 1.),
                        promotes_outputs=['r'])
submodel1.add_subsystem('sub1_ivc_theta', om.IndepVarComp('theta', pi),
                        promotes_outputs=['theta'])
submodel1.add_subsystem('subComp1', om.ExecComp('x = r*cos(theta)'),
                        promotes_inputs=['r', 'theta'],
                        promotes_outputs=['x'])

submodel2 = om.Group()
submodel2.add_subsystem('sub2_ivc_r', om.IndepVarComp('r', 2),
                        promotes_outputs=['r'])
submodel2.add_subsystem('sub2_ivc_theta', om.IndepVarComp('theta', pi/2),
                        promotes_outputs=['theta'])
submodel2.add_subsystem('subComp2', om.ExecComp('y = r*sin(theta)'),
                        promotes_inputs=['r', 'theta'],
                        promotes_outputs=['y'])

subprob1 = om.SubproblemComp(model=submodel1, inputs=['r', 'theta'],
                             outputs=['x'])
subprob2 = om.SubproblemComp(model=submodel2, inputs=['r', 'theta'],
                             outputs=['y'])

subprob1.add_design_var('r')
subprob1.add_design_var('theta')
subprob1.add_constraint('x')

subprob2.add_design_var('r')
subprob2.add_design_var('theta')
subprob2.add_constraint('y')

p.model.add_subsystem('sub1', subprob1, promotes_inputs=['r','theta'],
                      promotes_outputs=['x'])
p.model.add_subsystem('sub2', subprob2, promotes_inputs=['r','theta'],
                      promotes_outputs=['y'])
p.model.add_subsystem('supModel', model, promotes_inputs=['x','y'],
                      promotes_outputs=['z'])

p.setup(force_alloc_complex=True)

p.set_val('r', 1)
p.set_val('theta', pi)

p.run_model()
cpd = p.check_partials(method='cs')
print(f"x = {p.get_val('x')}")
print(f"y = {p.get_val('y')}") 
print(f"z = {p.get_val('z')}")

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
