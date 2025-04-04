[![Open in GitHub Codespaces](
  https://img.shields.io/badge/Open%20in%20GitHub%20Codespaces-333?logo=github)](
  https://codespaces.new/dwave-examples/3d-bin-packing?quickstart=1)
[![Linux/Mac/Windows build status](
  https://circleci.com/gh/dwave-examples/3d-bin-packing.svg?style=shield)](
  https://circleci.com/gh/dwave-examples/3d-bin-packing)

# 3D Bin Packing

Three-dimensional bin packing [[1]](#Martello) is an optimization problem where
the goal is to use the minimum number of bins to pack items with different
dimensions, weights and properties. Examples of bins are containers, pallets or
aircraft ULDs (Unit Load Device). 3D binpacking problems may include various
objectives and requirements. Basic requirements are boundary and geometric
constraints, which require that items be packed within bin boundaries and
without overlapping, respectively. There may be additional requirements on the
stability of the packing, flatness of top or bottom layers, fragility and weight
distribution, etc., which are not modeled in this formulation. In this example,
both items and bins are cuboids, and the sides of the items must be packed
parallel to the sides of bins.

This example demonstrates a formulation and optimization of a three-dimensional
multi bin packing problem using a
[constrained quadratic model](https://docs.dwavequantum.com/en/latest/concepts/models.html#constrained-quadratic-model) (CQM) that can be solved using a Leap hybrid
CQM solver.

![Demo Example](static/demo.png)

Below is an example output of the program:
<a id="plot"></a>
![Example Solution](_static/sample_solution_plot.png)

## Installation

You can run this example without installation in cloud-based IDEs that support
the
[Development Containers specification](https://containers.dev/supporting) (aka
"devcontainers") such as GitHub Codespaces.

For development environments that do not support `devcontainers`, install
requirements:

```bash
pip install -r requirements.txt
```

If you are cloning the repo to your local system, working in a
[virtual environment](https://docs.python.org/3/library/venv.html) is
recommended.

## Usage

Your development environment should be configured to access the
[Leap&trade; quantum cloud service](https://docs.dwavequantum.com/en/latest/ocean/sapi_access_basic.html).
You can see information about supported IDEs and authorizing access to your Leap
account
[here](https://docs.dwavequantum.com/en/latest/ocean/leap_authorization.html).

Run the following terminal command to start the Dash application:

```bash
python app.py
```

Access the user interface with your browser at http://127.0.0.1:8050/.

The demo program opens an interface where you can configure problems and submit
these problems to a solver.

Configuration options can be found in the [demo_configs.py](demo_configs.py)
file.

> [!NOTE]\
> If you plan on editing any files while the application is running, please run
the application with the `--debug` command-line argument for live reloads and
easier debugging: `python app.py --debug`

Alternatively, one can solve an instance of a problem through the terminal by
typing:

    python packing3d.py --data_filepath <path to your problem file>

There are several examples of problem instances under the `input` folder.

### Inputs

This is an example of a 3D bin packing input instance file with 1 bin and 35
cases.

```
# Max num of bins : 1
# Bin dimensions (L * W * H): 30 30 50

  case_id    quantity    length    width    height
---------  ----------  --------  -------  --------
0          12          5         3        8
1          9           12        15       12
2          7           8         5        11
3          7           9         12       4

```


Note that:
-   all bins are the same size.
-   there are several cases of each size (`quantity` sets the number of cases of
    identity `case id` with all having the dimensions defined in that row).

Run `python packing3d.py --help` in the terminal for a description of the demo
program's input parameters.

### Outputs

The program produces a solution like this:

```
# Number of bins used: 1
# Number of cases packed: 35
# Objective value: 104.457

  case_id    bin_location    orientation    x    y    z    x'    y'    z'
---------  --------------  -------------  ---  ---  ---  ----  ----  ----
        0               1              3   15    0    0     3     5     8
        0               1              3    9    0    0     3     5     8
        0               1              6    9   17    0     8     3     5
        0               1              3    0   25    0     3     5     8
        0               1              3   12    0    0     3     5     8
        0               1              2   20    0    0     5     8     3
        ...
```
Note that only a portion of the solution is shown above. Also, there are
multiple rows with same `case_id` as there are several items of the same size.
The number under the `orientation` column shows the rotation of a case inside a
bin as shown in the following figure.

![Orientations](_static/orientations.png)

The [graphic](#plot) at the top of the README is a visualization of this
solution.

Note that in this example, we do not model support or load-bearing constraints
for cases. Therefore, it is possible that larger cases are packed over smaller
cases without having full support.

## Problem Description

The goal of the 3D bin packing problem is to ensure that the cases are securely
packed within the fewest bins possible. The model sets the following objectives
and constraints to achieve this goal:

**Objectives:** minimize the height of the packed cases and number of bins used.

**Constraints:** the constraints for this problem fall into multiple categories.
- **Orientation constraints** focus on restricting cases to a single orientation.
- **Case and bin assignment constraints** ensure that each case is placed in
exactly one bin, that the chosen bin is in use, and that there are no gaps
between bins.
- **Geometric constraints** are used to make sure cases aren't assigned to
positions that would cause an overlap with another case.
- **Bin boundary constraints** enforce that cases are placed completely inside
their designated bin.

## Model Overview

In this example, to model multiple bins we assume that bins are located
back-to-back next to each other along x-axis of the coordinate system. This way
we only need to use one coordinate system with width of `W`, height of `H` and
length of `n * L` to model all bins,(see [below](#problem-parameters) for
definition of these parameters). That means that the first bin is located at
`0 <= x <= L`, second at `L < x <= 2 * L`, and last bin is located at
`(n - 1) * L < x <= n * L`. We apply necessary constraints to ensure that cases
are not placed between two bins.

### Problem Parameters

These are the parameters of the problem:

- `n`: number of bins
- `J`: set of bins `{1, ..., n}`
- `m` : number of cases
- `I`: set of cases `{1, ..., m}`
- `K` : possible orientations `{0, 1, ..., 5}`
- `Q` : possible relation (e.g., behind, above, etc) between every pair of cases
  `{0, 1, ..., 5}`
- `L` : length of the bins
- `W` : width of bins
- `H`: height of the bins
- `l_i`: length of case `i`
- `w_i`: width of case `i`
- `h_i`: height of case `i`

### Variables

- `v_j`:  binary variable that shows if bin `j` is used
- `u_(i,j)`:  binary variable that shows if case `i` is added to bin `j`
- `b_(i,k,q)`: binary variable defining geometric relation `q` between cases
  `i` and `k`
- `s_j`:  continuous variable showing height of the topmost case in bin `j`
- `r_(i,k)`: binary variable defining `k` orientations for case `i`
- `x_i`,`y_i`,`z_i`: continuous variable defining location of the back lower
  left corner of case `i` along `x`, `y`, and `z` axes of the bin

Note that we can determine some variables before the optimization. For example,
without loss of generality we can assign the first case to the first bin.
Therefore `u_{0,0} = 1` and `u_{0,j} = 0` for any `j\ne 0`.

We can also estimate a lower bound on the number of occupied bins:

![jmin](_static/jmin.png)

The first `jmin` bin variables `v_j` can therefore be set equal to 1 without
loss of generality. In what follows, the set of bins `J` is `{jmin, ..., n}`.

### Expressions

Expressions are linear or quadratic combinations of variables used for easier
representations of the models.

- `x'_i`,`y'_i`,`z'_i`: effective length, width and height of case `i`,
  considering orientation, along `x`, `y`, and `z` axes of the bin
- `o_1`, `o_2`, `o_3`: terms of the objective

### Objective

Our objective contains three terms:

The first term is to minimize the average height of the cases in a bin which
ensures that cases are packed down.

![eq1](_static/eq1.png)

The second term in the objective ensures that the height of the topmost case in
each bin is minimized. This objective is weakly considered in the first
objective term, but here is given more importance.

![eq2](_static/eq2.png)

Our third objective function minimizes the total number of the bins.

![eq3](_static/eq3.png)

Note that we multiplied this term by the height of the bins so its contribution
to the objective has same weights as the first and second objective terms.
The total objective value is the summation of all these terms.

### Constraints

#### Orientation Constraints

Each case has exactly one orientation:

![eq4](_static/eq4.png)

Orientation defines the effective length, width, and height of the cases along
`x`, `y`, and `z` axes.

![eq5](_static/eq5.png)

![eq6](_static/eq6.png)

![eq7](_static/eq7.png)

#### Case and Bin Assignment Constraints

Each case goes to exactly one bin.

![eq8](_static/eq8.png)

Only assign cases to bins that are in use.

![eq9](_static/eq9.png)

Ensure that bins are added in order; i.e., bin `j` is in use
before bin `j + 1`.

![eq10](_static/eq10.png)

#### Geometric Constraints

Geometric constraints, required only for three-dimensional problems, are applied
to prevent overlaps between cases. In the following constraints, "left" and
"right" refer to the position of the case along the `x` axis of a bin, "behind"
and "in front of" to the `y` axis, and "above" and "below" to the `z` axis.
To avoid overlaps between each pair of cases we only need to ensure that at
least one of these situations occur:

- case `i` is on the left of case `k` (`q = 0`):

![eq11](_static/eq11.png)

- case `i` is behind case `k` (`q = 1`):

![eq12](_static/eq12.png)

- case `i` is below case `k` (`q = 2`):

![eq13](_static/eq13.png)

- case `i` is on the right of case `k` (`q = 3`):

![eq14](_static/eq14.png)

- case `i` is in front of case `k` (`q = 4`):

![eq15](_static/eq15.png)

- case `i` is above case `k` (`q = 5`):

![eq16](_static/eq16.png)

To enforce only one of the above constraints we also need:

![eq17](_static/eq17.png)

#### Bin Boundary Constraints:

These sets of constraints ensure that case `i` is not placed outside the bins.

![eq18](_static/eq18.png)

![eq19](_static/eq19.png)

![eq20](_static/eq20.png)

![eq21](_static/eq21.png)

When `u_{i,j}` is 0 these constraints are relaxed.

Note that the width constraints do not involve the variables `u_{i,j}`. This is
because the bins are virtually placed along the x-axis and all the cases have to
satisfy the width constraints in a similar manner. Similar considerations apply
for the height constraints, however, here we want to track the bin height `s_j`.

## References

<a id="Martello"></a>
[1] Martello, Silvano, David Pisinger, and Daniele Vigo.
"The three-dimensional bin packing problem."
Operations research 48.2 (2000): 256-267.

## License

Released under the Apache License 2.0. See [LICENSE](LICENSE) file.