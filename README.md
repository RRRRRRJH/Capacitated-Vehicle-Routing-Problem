

# Capacitated Vechile Routing

Multiobjective python implementation of capacitated vehicle routing
 problem with NSGA-II algorithm using [deap](https://github.com/deap/deap) package.


## Contents
- [Installation](#installation)
    - [Requirements](#requirements)
    - [Installing with Virtualenv](#installing-with-virtualenv)
- [Problem Statement](#problem-statement)
- [Parsing Input](#parsing-input)
    - [Text File Format](#text-file-format)
    - [JSON Format](#json-format)
    - [Convert `*.txt` to `*.json`](#convert-txt-to-json)
- [Algorithm Selection](#algoritm-selection)
- [Assumptions](#assumptions)
- [NSGA-II Implementation](#nsga-ii-implementation)
    - [Individual (Chromosome)](#individual-chromosome)
        - [Individual Coding](#individual-coding)
        - [Individual Decoding](#individual-decoding)
        - [Print Route](#print-route)
    - [Fitness evaluation](#fitness-evaluation)
    - [Selection: Non dominated sorting selection](#selection-non-dominated-sorting-selection)
    - [Crossover: Ordered Crossover](#crossover-ordered-crossover)
    - [Mutation: Inverse Operation](#mutation-inverse-operation)
    - [Algorithm](#algorithm)
- [Running Tests](#running-tests)
- [Visualizations](#visualizations)
- [File Structure](#file-structure)
- [Framework Documentation](#framework-documentation)
- [Future Improvements](#future-improvements)
- [License](#license)


## Installation
### Requirements
- [Python 3.8](https://docs.python.org/)
- [Pip](https://pypi.python.org/pypi/pip)
- [Virtualenv](https://virtualenv.pypa.io/en/stable/)

### Installing with Virtualenv
On Unix, Linux, BSD, macOS, and Cygwin:

```sh
git clone https://github.com/icav_vrp.git
cd icav_vrp
virtualenv --python=python3 venv
source venv/bin/activate
pip install -r requirements.txt
```

## Problem Statement
Delivery companies every day
need to deliver packages to many different clients. The deliveries are accomplished using an
available fleet of vehicles from a central warehouse. The goal of this exercise is to design a
route for each vehicle so that all customers are served, and the number of vehicles
**(objective 1)** along with the total traveled distance **(objective 2)** by all the vehicles are
minimized. In addition, the capacity of each vehicle should not be exceeded (constraint 1)

**Notes**
1. For each client we have we only need to consider first four fields. They are `id, xcoord, ycoord, demand`
2. Distance between each client can be calculated using `Euclidian formula`



## Parsing Input

### Text File Format
The text files for this problem which are inputs provided can be found in the 
`data/text` directory. Text file is named as `Input_Data.txt`.

The description of the format of the text file defined in the problem instance

```
<Instance Name>
<empty line>
VEHICLE
NUMBER  CAPACITY
 K        Q
<empty line>
CUSTOMER
CUST NO.   XCOORD.  YCOORD.    DEMAND    READY TIME  DUE DATE  SERVICE TIME
 
    0      40         50          0          0       1236          0   
    1      45         68         10          0       1127         90   
    2      45         70         30          0       1125         90
    .......
    n      x_n        y_n       d_n         r_n      due_n        s_n 
```

**Definitions**
1. `CUST No.` 0 denotes the depot , where all vehicles start and finish
2.  `K` denotes maximum number of vehicles that are available
3.  `Q` denotes maximum capacity of each vehicle
4.  `n` denotes the maximum number of customers in the given problem, excluding the depot

### JSON Format
Since text file is pretty bad to behave in object oriented way , we are converting
given text input file into JSON format. And they are stored under `data/json` directory.
The file name of these json file is same as text file and it can be named as instance in the
code. Since our problem is `Input_Data.txt` our json file will be `Input_Data.json`.

**Notes**
1. We are adding additional fields here such as `Number_of_customers`,`distance_matrix`
which are calculated from `Input_Data.txt`.
2. `distance_matrix` contains distances from each customer to other with first array being distances from
depot.
3. If a json file doesn't exist at the `data/json` directory , when we run `parseText2Json` it creates
json file. If it exists its overwritten.


Below is description of the JSON format.

```js
{
    "instance_name" : "<Instance name>",
    "Number_of_customers" : n,
    "max_vehicle_number" : K,
    "vehicle_capacity" : Q,
    "depart" : {
        "coordinates" : {
            "x" : x0,
            "y" : y0
        },
        "demand" : q0,
        "ready_time" : e0,
        "due_time" : l0,
        "service_time" : s0
    },
    "customer_1" : {
        "coordinates" : {
            "x" : x1,
            "y" : y2
        },
        "demand" : q1,
        "ready_time" : e1,
        "due_time" : l1,
        "service_time" : s1
    },
    ...
    "customer_n" : {
        "coordinates" : {
            "x" : x_n,
            "y" : y_n
        },
        "demand" : q100,
        "ready_time" : e100,
        "due_time" : l100,
        "service_time" : s100
    },
    "distance_matrix" : [
        [dist0_0, dist0_1, ..., dist0_n],
        [dist1_0, dist1_1, ..., dist1_n],
        ...
        [distn_0, distn_1, ..., dist0_0]
    ]
}

```

### Convert `*.txt` to `*.json`
Run the `parseText2Json.py` to convert `*.txt` file to `*.json` file.

```sh
python parseText2Json.py
```
## Algorithm Selection
Multiobjective optimization of travelling salesman is a NP-hard problem.
So simple Genetic algorithm cannot compute good solutions when there are
multiple objectives. We need a non dominated sorting approach where only
non dominated inviduals are selected. Another way of formulating this problem is to 
use genetic algorithm but combining both objectives into a single one.

Here our objective is 
```
Minimize -> Number of vehicles
Minimize -> Distance travelled by all vehicles
```

This can be formulated in another way like this
```
Minimize -> (Number of vehicles) * (Distance travelled by all vehicles)

---- or -----

Maximize -> 1/ (Num of vehicles) *(Distance by all )
```

## Assumptions
We are assuming the following things.

1. There is no time delay and no time windows for our vehicle at the objective locations
2. Fixed cost for extra vehicle is assumed to be 0.
3. Due date, service time , ready time are Ignored
4. Distance between client to client is assumed to be Euclidean.
5. Vehicle always starts from the depot `customer_0` and delivers goods 
and then comes back to depot again after delivery
6. 


## NSGA-II Implementation

### Individual (Chromosome)
#### Individual Coding
All visited customers of a route (including several sub-routes) are coded into an `individual` in turn. For example, the following route

```
Sub-route 1: 0 - 5 - 3 - 2 - 0
Sub-route 2: 0 - 7 - 1 - 6 - 9 - 0
Sub-route 3: 0 - 8 - 4 - 0
```
are coded as `5 3 2 7 1 6 9 8 4`, which can be stored 
in a Python `list` object, i.e., `[5, 3, 2, 7, 1, 6, 9, 8, 4]`.


#### Individual Decoding
The route of is given as sequence of customers , but it is actually divided in to subroutes according to the
load carrying capacity of the vechicle. Starting from the depot demand from each customer is added and when the load
exceeds the vechicle capacity the subroute is closed and assigned to that vechicle. This process is repeated until all
the customers in the sequence is fulfilled. Thus we will get subroutes from routes and number of subroutes
is equivalent to number of vechicles.

```python
routeToSubroute(individual, instance)
```
Decodes an `individual` to `route`. Refer the below example

```python
# Individual
[12, 14, 16, 8, 9, 11, 21, 20, 5, 3, 4, 6, 10, 7, 2, 1, 22, 25, 24, 23, 18, 19, 17, 15, 13]

# Route
[[12, 14, 16], [8, 9, 11, 21, 20], [5, 3, 4, 6, 10], [7, 2, 1], [22, 25, 24], [23, 18, 19, 17], [15, 13]]
```

### Fitness Evaluation
Since our problem is Multiobjective , we need to calculated two objectives here
One is `Number of vehicles` and `Total Distance Travelled`

So we divided the objectives, calculate them and return a `tuple`

First objective is calculated using `getNumVehiclesRequired` function
which just finds number of elements in list after the `Indiviudal` is 
passed in to `routeToSubroute`

Second objective is caclulated using `getRouteCost`.
After dividing an individual in to sub routes, For each subroute distance
is calculated between indiviudals and added. Final Route cost will be
addition of all these sub routes cost

```python
eval_indvidual_fitness(individual, instance, unit_cost)
```
This function returns a tuple of `(Num of vehicles, Route Cost)`
We have to minimize both the objectives at same time.
So when we create our individual using [deap]() package, we have to specify the
individual in following way - 

```python
creator.create('FitnessMin', base.Fitness, weights=(-1.0, -1.0))
```

Assigning weights (-1.0, -1.0) is crucial step when defining objective.

### Selection: Non dominated sorting selection

### Crossover: Ordered Crossover

### Mutation: Inverse Operation

### Algorithm


## Running Tests
We used python inbuilt `unittest` module to run all the tests
To run all the tests do the following

```sh
python -m unittest discover test
```
This command will discover all the tests in the test folder and runs them all.


## Visualizations

## File Structure

## Framework Documentation

## Future Improvements

## License





























