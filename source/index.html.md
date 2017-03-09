---
title: Enigma Documentation

language_tabs:
  - python: Python
  - shell: Shell

toc_footers:
  - <a href='#'>Sign Up for a Developer Key</a>
  - <a href='https://github.com/tripit/slate'>Documentation Powered by Slate</a>

search: true
---

# Introduction

Enigma is a distributed, P2P cloud storage and computation platform for encrypted data.

This documentation will cover deploying an Enigma instance, setting up a client, storing data, setting access permissions, and executing computations.

# Deployment

## Start Seed Node

```shell
docker run enigma -seed 7733
```

Start an Enigma instance that facilitates peer discovery.

`docker run enigma -seed <port>`

Parameter | Type  | Description
--------- | ----- | -----------
`port`    | `int` | local port of seed node

## Add Node

Starts a new Enigma instance and connects it to an existing seed node.

```shell
docker run enigma 24.133.21.31 7733
```

`docker run enigma <ip> <port>`

Parameter | Type     | Description
--------- | -------- | -----------
`ip`      | `string` | ip address of the seed node
`port`    | `int`    | port of the seed node

# Enigma Architecture

Here we provide a brief overview of the various components of Enigma's platform, describing the core data types, data keys, identities, and access control.

## Data Types

Enigma is able to perform computation over two basic types of numbers: integers and fixed-point numbers.
Fixed point numbers include a finite range of precision that is probabilistically rounded during truncation.
Our framework then also supports vectors and matrices comprised of these scalar types.

Type               | Description
------------------ | -----------
`enigma.Type.Int`  | integer scalar
`enigma.Type.IntV` | integer vector
`enigma.Type.IntM` | integer matrix
`enigma.Type.FP`   | fixed-point scalar
`enigma.Type.FPV`  | fixed-point vector
`enigma.Type.FPM`  | fixed-point matrix

**Related**

* [Access Control](#access-control)
* [Arithmetic](#arithmetic)
* [Comparisons](#comparisons)
* [Statistics](#statistics)

<aside class="notice">
  Keep in mind that all numbers are taken modulo a large prime, usually greater than 64 bits.
  As such, it is still possible to cause overflows, as in standard integer types.
</aside>

## Keys 

Data in an Enigma network is secret-shared to multiple nodes for privacy.
After a value has been stored, it is referenced by a data *key*.
We will use the notation `x:Type` to denote the key `x` whose underlying type is `Type`.
Enigma has two distinct types of keys: standard keys and table keys.

### Standard Keys

A *standard key* is a string used to store and retrieve values in a privacy-preserving, key-value store.
Keys may be chosen by the user, but by default are generated at random like the one shown below.
This helps to prevent any leakage about the underlying values that can be determined by the key.

`key = '2b55fadde2424b168d03dad2b02b5816228260bf5d3c7855f05d811be54d31c2'`

### Table Keys

A *table key* is a string that allows a user to select one or more columns from a table, like those found in relational databases.
Table and column names may be chosen by the user, and have a corresponding type dictated by the table's schema.  

Table keys can be used to select one or more columns, as follows:

Key Format               | Type            | Description
------------------------ | --------------- | -----------
`'Table.col1'`           | `IntV` or `FPV` | selects the vector `col1` from `Table`
`'Table.col1,col2,col3'` | `IntM` or `FPM` | selects the matrix `[col1,col2,col3]` from `Table`

**Related**

* [Key-Value Storage](#key-value-storage)
* [Table Storage](#table-storage)

<aside class="notice">
  Note that for privacy and performance reasons, the client does not maintain metadata or data keys for values the user has stored.
  These must be persisted elsewhere if they are to be used across multiple sessions.
</aside>

## Identity

Enigma recognizes two types of identities, *users* and *groups*.

### Users

An Enigma user is identified and authenticated using a secp256k1 key pair.
The public key serves as an identifier for a particular user, while the secret key is used to encrypt and authenticate commands from the client.
All data in Enigma has an explicit owner (a user or a group), thus the [Client](#client) persists secret keys so that one's data can be accessed across sessions.

### Groups

A group is a named collection of users, identified by their public keys.
The user who creates a group becomes its owner, allowing them to modify group membership and access controls for the data owned by a group.

**Related**

* [Key Management](#key-management)
* [Making Groups](#making-groups)

## Access Control

Each piece of data is owned either by a user or a group.
In order to run computations on a given key, the requesting user must minimally be either the data's owner, or a member of the group that owns the data.

Enigma also allows the owner of a piece of data to whitelist a set of operations for which it can be used an input.
If a piece of data is owned by a group, only the creator of the group will be able to edit the whitelist.

**Related**

* [Allowing Operations](#allowing-operations)

<aside class="notice">
  As this is a beta, the current default for an empty whitelist is to allow any operation.
  This may change in the future for obvious security reasons.
</aside>


# Client Initialization


```python
import enigma

with enigma.Client(vaddr, key_path, explain=True) as client:
  # Automatically cleans up worker threads and network calls
```

`Client(vaddr, key_path, explain=True)`

A client is used to issue authenticated commands for performing data storage, permissioning, access controls, and computation.
Initializing the client requires the public address of the validator and a file path to a user's secret key.
If no such file exists, the client will automatically generate and write a new public key to the file, which can be used to resume future sessions.
The optional `explain` parameter will cause the client to describe its actions, which may be useful for debugging.
Instantiating the client will cause it to connect and register with the validator, and can then be used to issue commands.

Parameter   | Type         | Description
----------- | ------------ | -----------
`vaddr`     | `string`     | address of validator, `'ip:port'`
`key_path`  | `string`     | path to the user's key file
`explain`   | `bool=False` | flag indicating whether the client should print information about it's execution


# Key Management

## Private Keys

```python
client.get_privkey()
#> u'a1d1fe5e45a433580997c77eeedfc0ed80d2905f443a0f5ffadd4c0f155097e3'
```

`Client#get_privkey()`

Get the client's private key.

Parameter   | Type      | Description
----------- | --------- | -----------
**returns** | `unicode` | the client's hex-encoded secp256k1 private key


## Public Keys

```python
client.get_pubkey()
#> u'030662b1f5d7cac1083629a007df44d854967c1d8bc541854e67d48d7e635367bd'
```

`Client#get_pubkey()`

Get the client's public key.

Parameter   | Type      | Description
----------- | --------- | -----------
**returns** | `unicode` | the client's hex-encoded, compressed secp256k1 public key

```python
client.get_user()
#> u'030662b1f5d7cac1083629a007df44d854967c1d8bc541854e67d48d7e635367bd'
```

`Client#get_user()`

Get the client's current user, alias for `Client#get_pubkey()`.

Parameter   | Type      | Description
----------- | --------- | -----------
**returns** | `unicode` | the client's hex-encode, compressed secp256k1 public key

**Related**

* [Identity](#identity)
* [Making Groups](#making-groups)

# Permissions

## Making Groups

```python
name = 'boston hospitals'
others = [user1, user2]

# create group called `name` with members `[client.get_user(), user1, user2]`
client.group(name, others)
#> True
```

`Client#group(name, others)`

Creates a named group consisting of the current user and the provided list of other users.
The current user becomes the group's owner.

Parameter   | Type     | Description
----------- | -------- | -----------
`name`      | `string` | name of the group
`others`    | `list`   | list of users other than the current user
**returns** | `bool`   | success

**Related**

* [Identity](#identity)
* [Public Keys](#public-keys)

## Allowing Operations

```python
import enigma.OpCode

keys = ['Patients.age', 'Patients.height', 'Patients.weight']
ops = [OpCode.LinReg, OpCode.Var, OpCode.Mean]

client.allow(keys, ops)
#> True
```

`Client#allow(keys, ops)`

Whitelists the set of operations for each of the given data keys.

Parameter   | Type   | Description
----------- | ------ | -----------
`keys`      | `list` | set of data keys
`ops`       | `list` | set of OpCodes to be whitelisted
**returns** | `bool` | success



**Related**

* [Data Types](#data-types)
* [Keys](#keys)
* [Arithmetic](#arithmetic)
* [Comparisons](#comparisons)
* [Statistics](#statistics)

## OpCodes 

OpCode          | Operations                   | Reference
--------------- | ---------------------------- | ---------
`OpCode.Sum`    | `sum`                        | [Addition](#addition)
`OpCode.Mul`    | `mul`                        | [Multiplication](#multiplication)
`OpCode.Dot`    | `dot`                        | [Dot](#dot)
`OpCode.Eq`     | `eq`                         | [Equality](#equality)
`OpCode.Cmp`    | `gt`, `gte`, `lte`, and `lt` | [Comparisons](#comparisons)
`OpCode.Mean`   | `mean`                       | [Mean](#mean)
`OpCode.Var`    | `var`                        | [Variance](#variance)
`OpCode.LinReg` | `linreg`                     | [Linear Regression](#linear-regression)

# Key-Value Storage

## Store Data

```python
# Store integer scalar, vector, and matrix
k1 = client.store(3)
k2 = client.store([1,2,3])
k3 = client.store([[1,0], [0,1]])

# Store fixed-point scalar, vector, and matrix
k1 = client.store(3.0)
k2 = client.store([1.0, 2.0, 3.0])
k3 = client.store([[1.0,0.0], [0.0,1.0]])

# Store value accessible to group 'enigma-dev'
k1 = client.store(21, group='enigma-dev')
#> k1:'82049950f7ac44b69b690e2913d084c5086736f3a6a26bf7add4361a36531bb1'

# Store value with user chosen key
client.store(21, key=k1)
#> '82049950f7ac44b69b690e2913d084c5086736f3a6a26bf7add4361a36531bb1'


# If at least one element is of type `float`,
# all elements are promoted to `float`.

# stores a fixed-point vector
k2 = client.store([1.0, 2, 3])

# stores a fixed-point matrix
k3 = client.store([[1.0,0], [0,1])
```

`Client#store(value, key=None, group=None)`

Stores the provided value under the provided standard key.

Parameter   | Type                      | Description
----------- | ------------------------- | -----------
`value`     | `int`, `float`, or `list` | scalar, vector, or matrix of `int` or `float` values, also accepts `numpy.array`
`key`       | `string=None`             | desired key under which to store `value`, otherwise a random key is generated
`group`     | `string=None`             | name of the group to own `value`, otherwise the user will solely own the data
**returns** | `string` or `None`        | if successful, the key under which `value` is stored, otherwise `None`

<aside class="success">
  The underlying type of `value` will be inferred based on the input provided.
  For multidimensional types, if any of the elements are `float`, the entire vector or matrix will be lifted into `FP*`.
</aside>

## Load Data

```python
key = client.store([1,2,3])

client.load(key)
#> [1,2,3]
```

`Client#load(key)`

Retrieves a value from the network, compatible with standard and table keys.

Parameter   | Type                      | Description
----------- | ------------------------- | -----------
`key`       | `string`                  | key of the value to load
**returns** | `int`, `float`, or `list` | plaintext scalar, vector or matrix

# Table Storage

## Making Schemas

```python
enigma.Schema([
  ('age',    enigma.Type.Int), # years
  ('height', enigma.Type.FP),  # inches
  ('weight', enigma.Type.FP),  # pounds
])
#> (age:Int, height:FP, weight:FP)
```

`Schema(schema_desc)`

Creates a table schema, specifying the name and type of each field.
The schema is constructed from a list of `(str, enigma.Type)` tuples.

Parameters    | Type          | Description
------------- | ------------- | -----------
`schema_desc` | `list`        | list of tuples, each consisting of a column's name and type
**returns**   | `Schema` | map that preserves the ordering of columns in `schema_desc`

## Creating Tables

```python
client.create_table('Patients', schema, 'boston hospitals')
#> True
```

`Client#create_table(name, schema, group=None)`

Parameters    | Type          | Description
------------- | ------------- | -----------
`name`        | `string`      | name of the table
`schema`      | `schema`      | map of field names to the `enigma.Type` of the column
`group`       | `string=None` | name of the group to own `value`, otherwise the user will solely own the data
**returns**   | `bool`        | success

## Inserting Rows

```python
client.insert('Patients', [
  [18, 64.3, 118.2],
  [35, 73.8, 180.6],
  [24, 69.5, 210.7],
])
#> 3
```

`Client#insert(name, rows)`

Inserts a set of rows into the table with the provided name.

Parameters  | Type     | Description
----------- | -------- | -----------
`name`      | `string` | name of the table
`rows`      | `list`   | row-oriented matrix consisting of the values to be inserted
**returns** | `int`    | number of rows inserted

<aside class="success">
  As with key-value storage, the client will attempt to perform type promotion on the provided rows.
  However, the promotion is done on a per-column basis, as each column is treated independently.
  Furthermore, if the client does not have enough information to do type promotion, nodes will promote `IntV` columns to `FPV` in accordance with the table's schema.
</aside>


# Arithmetic

## Addition

```python
# summing integer scalars
k1 = client.store(3)
k2 = client.store(4)
client.sum(k1, k2)
#> 7

# summing integer vectors
k1 = client.store([1, 3])
k2 = client.store([2, 1])
client.sum(k1, k2)
#> [3, 4]

# summing integer matrices
k1 = client.store([[1,0],[0,1]])
k2 = client.store([[0,1],[1,-1]])
client.sum(k1, k2)
#> [[1,1], [1,0]]
```

`Client#sum(x:Type, y:Type)`

Computes the sum of two identical data types.

Parameters  | Type                      | Description
----------- | ------------------------- | -----------
`x`         | `string`                  | key for value of type `Type`
`y`         | `string`                  | key for value of same type `Type`
**returns** | `int`, `float`, or `list` | sum of `x` and `y`

## Multiplication

```python
# multiplying integer scalars
k1 = client.store(3)
k2 = client.store(4)
client.mul(k1, k2)
#> 12

# point-multiplying integer vectors
k1 = client.store([1, 3])
k2 = client.store([2, 1])
client.mul(k1, k2)
#> [2, 3]

# point-multiplying integer matrices
k1 = client.store([[1,0], [0,1]])
k2 = client.store([[0,1], [1,-1]])
client.mul(k1, k2)
#> [[0,1], [1,-1]]
```

`Client#mul(x:Type, y:Type)`

Computes the point-wise multiplication of two identical data types.

Parameters  | Type                      | Description
----------- | ------------------------- | -----------
`x`         | `string`                  | key for value of type `Type`
`y`         | `string`                  | key for value of same type `Type`
**returns** | `int`, `float`, or `list` | point-wise multiplication of `x` and `y`

## Dot

```python
# dotting integer vectors
k1 = client.store([1, 3])
k2 = client.store([2, 1])
client.dot(k1, k2)
#> 5
```

`Client#dot(x:Type, y:Type)`

Computes the dot product of two identical data types.

Parameter   | Type                      | Description
----------- | ------------------------- | -----------
`x`         | `string`                  | A key of type `IntV` or `FPV`.
`x`         | `string`                  | A key of type `Int` or `FPV`.
**returns** | `int`, `float`, or `list` | dot product of `x` and `y`

# Comparisons

## Greater Than

```python
client.gt(client.store(4), client.store(3))
#> True
client.gt(client.store(3), client.store(3))
#> False
```

`Client#gt(x:Int, y:Int)`

Parameter   | Type     | Description
----------- | -------- | -----------
`x`         | `string` | A key for an `Int` value.
`y`         | `string` | A key for an `Int` value.
**returns** | `bool`   | Returns x > y.

## Greater Than or Equal

```python
client.gte(client.store(3), client.store(4))
#> False
client.gte(client.store(4), client.store(3))
#> True
client.gte(client.store(3), client.store(3))
#> True
```

`Client#geq(x:Int, y:Int)`

Parameter   | Type     | Description
----------- | -------- | -----------
`x`         | `string` | A key for an `Int` value.
`y`         | `string` | A key for an `Int` value.
**returns** | `bool`   | Returns x >= y.

## Equality

```python
client.eq(client.store(3), client.store(3))
#> True
```

`Client#eq(x:Int, y:Int)`

Parameter   | Type     | Description
----------- | -------- | -----------
`x`         | `string` | A key for an `Int` value.
`y`         | `string` | A key for an `Int` value.
**returns** | `bool`   | Returns x == y.

## Less Than or Equal

```python
client.leq(client.store(4), client.store(3))
#> False
client.leq(client.store(3), client.store(4))
#> True
client.leq(client.store(3), client.store(3))
#> True
```

`Client#leq(x:Int, y:Int)`

Parameter   | Type     | Description
----------- | -------- | -----------
`x`         | `string` | A key for an `Int` value.
`y`         | `string` | A key for an `Int` value.
**returns** | `bool`   | Returns x <= y.

## Less Than

```python
client.lt(client.store(3), client.store(4))
#> True
client.lt(client.store(3), client.store(3))
#> False
```

`Client#lt(x:Int, y:Int)`

Parameter   | Type     | Description
----------- | -------- | -----------
`x`         | `string` | A key for an `Int` value.
`y`         | `string` | A key for an `Int` value.
**returns** | `bool`   | Returns x < y.

# Statistics

## Mean

```python
v = client.store([1,4,9])
client.mean(v)
#> 4.6667
```

`Client#mean(v:FPV)`

Computes the mean of a vector of fixed-point numbers.

Parameter   | Type     | Description
----------- | -------- | -----------
`v`         | `string` | A key for an `FPV` value.
**returns** | `float`   | Returns the mean of `v`.

## Variance

```python
v = client.store([1.0,4,9])
client.var(v)
#> 100.62839
```

`Client#var(v:FPV)`

Computes the variance of a vector of fixed-point numbers.

Parameter   | Type     | Description
----------- | -------- | -----------
`v`         | `string` | A key for an `FPV` value.
**returns** | `float`  | Returns the variance of `v`.

## Linear Regression

```python
Akey = client.store(A)
bkey = client.store(b)
x0key = client.store(x0)

client.linreg(Akey, bkey, x0key)
#> [740.324745]
```

`Client#linreg(A:FPM, b:FPV, x0:FPV)`
  
Performs linear regression of a matrix of independent parameters and their relation to some dependent target vector, using an initial approximation.

Parameter   | Type     | Description
----------- | -------- | -----------
`A`         | `string` | key for an `n` by `m` fixed-point matrix, the independent variables.
`b`         | `string` | key for a length `n` fixed-point vector, the dependent variables.
`x0`        | `string` | key for a length `m` fixed-point vector, the initial approximation.
**returns** | `list`   | values representing the solution's coefficients.

# Errors

Status Code | Meaning | Description
----------- | ------- | -----------
**1x** | **Success** |
`10` | OK | The request was successfully processed.
`11` | Created | The requested resource has been successfully created.
   | 
**2x** | **Request Failure** |
`20` | Bad Request | The request is malformed, error message returned.
   |
**3x** | **Permission Failure** |
`30` | Unauthenticated | The requesting user was not successfully authenticated.
`31` | Unauthorized | The requesting user was not authorized to perform the request.
   |
**4x** | **Resource Failure** |
`40` | Not Found | At least one of the resources required to process the request was not found.
`41` | Data Inconsistency | The persisted resource has been corrupted or is otherwise inconsistent.
`42` | Storage Unavailable | The persistent storage required to process the request is unavailable.
`43` | Network Unavailable | At least one participant was or became unreachable while processing the request.
`44` | Too Many Requests | The client is being rate limited.
   |
**5x** | **Execution Failure** |
`50` | Aborted | The requested operation aborted.
`51` | Internal Server Error | The request could not be processed successfully.

# Advanced Deployment

An Enigma network consists of two types of processes--a single *validator* and `N` *nodes*.
The validator is responsible for disseminating network parameters and ensuring the correctness of MPC executions.
Nodes are responsible for storing secret-shared data and collaboratively executing commands from clients.
We currently provide a Docker image called `enigma:0.1` that contains a precompiled executable called `enigma`.
For installation instructions, see [enigma-docker](https://github.com/enigmampc/enigma-docker).

## Key Files

```shell
enigma keygen [-keyname|-force]
```

```json
{
  "privkey": "5b6a6d44b9d4b398332625c9690bd5ecfe9d7a0a01875a4057fdcc0071b02593",
  "pubkey": "03fa726ea7bbb81cbe3571032567c3a27796b9786b34c7910936897cb715f7d341"
}
```

All communication in Enigma is encrypted and authenticated using secp256k1 keypairs.
Each instance (node or validator) should have its own distinct keypair, which can be loaded from disk during startup.
The default `keyname` is `enigma`, which will write the key to file called `enigma.key` in the current directory.
If multiple processes are running in the same directory, it will be useful to name the keys accordingly, e.g. `validator.key, node1.key, node2.key, ...`.

Optional | Type   | Description
-------- | ------ | -----------
-keyname | string | the name of the key file, the final path is suffixed with `".key"`
-force   | flag   | if present, overwrites any existing key file with the same name

## Configuration Files

```shell
enigma configure -vaddr <pub-addr> [-keyname] [-confname|-force] [-N|-T|-E|-F|-P]
```

```json
{
  "params": {
    "validator": {
      "pubkey": "03fa726ea7bbb81cbe3571032567c3a27796b9786b34c7910936897cb715f7d341",
      "addr":   ":7733"
    },
    "n": 3,
    "t": 2,
    "e": 48,
    "f": 16,
    "p": 22953686867719691230002707821868552601124472329079
  }
}
```

During start up, each validator or node reads from a configuration file that specifies critical network parameters.
Most importantly, it contains the validator's address and public key, which allows nodes to register themselves with the validator and receive membership updates.
These parameters are also disseminated to clients to ensure they can securely communicate with the network and properly generate secret shares.

Parameter   | Type             | Description
----------- | ---------------- | -----------
`-vaddr`    | `string=:7733`   | the validator's TCP address
`-keyname`  | `string=enigma`  | name of the validator's key
`-confname` | `string=enigma`  | name of the configuration file, the final path is suffixed by `.conf`
`-force`    | `flag`           | if present, overwrites any existing key file with the same name
`-N`        | `int=3`          | number of nodes in the network
`-T`        | `int=2`          | secret-share threshold for reconstruction
`-E`        | `int=48`         | number of integral bits in a fixed-point number
`-F`        | `int=16`         | number of precision bits in a fixed-point number
`-P`        | `base-10-string` | secret-share modulus, must be a large prime with more than `2*(E+F)` bits

If the instances will be running on separate machines or containers, please ensure that `-vaddr` specifies the validator's *public address*, that is, one that is reachable from the other machines or containers.
The default is `:7733`, which is only useful for local Enigma networks.

The configuration step requires reading a public key from an existing [key file](#key-files).
If a `-keyname` is provided, it will read from `<name>.key`, otherwise defaulting to `enigma.key`.

If successful, the configuration will be written to `enigma.conf` (or `<name>.conf` if `-confname` was specified). 
Again, the `-force` flag can be used to overwrite an existing configuration file.

### Installing Configuration Files

```shell
enigma printconfig [-confname]
```

We have also provided a `printconfig` command which prints the contents of a configuration file as a single line to the terminal.
This will be useful for installing identical configuration files on different machines.

Parameter   | Type            | Description
----------- | --------------- | -----------
`-confname` | `string=enigma` | name of the configuration file to print

Once the output has been printed and copied, it can be installed using `echo '<output>' > enigma.conf`, or to some other configuration file name.
Though the output of `printconfig` is a single line, the JSON will be reformatted and persisted after being loaded from disk by a node or validator.

## Starting Network

### Validator

```shell
enigma start validator -keyname enigma.key -confname enigma.conf
```

`enigma start validator [-keyname|-confname]

The validator will listen on the address specified by the configuration file.
By default, `enigma.key` and `enigma.conf` will be used unless otherwise specified.

Parameter   | Type            | Description
----------- | --------------- | -----------
`-keyname`  | `string=enigma` | name of the key file
`-confname` | `string=enigma` | name of the configuration file

### Nodes

```shell
enigma start node -naddr 23.173.82.12:7733 -keyname enigma.key -confname enigma.conf
```

`enigma start node -naddr <pub-addr> [-keyname|-confname]

On startup, the node will attempt to connect to the validator address in the configuration file.
Please ensure that `-naddr` is the public address of the node.
This address will be registered with the validator and disseminated to other nodes and clients that connect to the network.

Parameter   | Type            | Description
----------- | --------------- | -----------
`-naddr`    | `string`        | listening address of the node
`-keyname`  | `string=enigma` | name of the key file
`-confname` | `string=enigma` | name of the configuration file

<aside class="notice">
During startup, each instance will actually begin listening on two different ports.
The primary port (specified by `-vaddr` or `-naddr`) is used for traffic inside the Enigma network. 
The second port, listening on primary port +1, is used for communicating with clients. 
The reason is twofold, to separate security concerns and also because they utilize different encoding schemes. 
This information is not necessary, but may be useful in debugging any potential issues that may arise during setup. 
Given the validator's and node's primary addresses, the client is designed to automatically establish connections over their respective client ports.
</aside>

## Walkthrough

```shell
# SETUP SEED NODE
docker run -it --name validator -p 7733:7733 -p 7734:7734 enigma:0.1
enigma keygen
enigma configure -vaddr <validator-pub-ip>:7733
enigma printconfig
# Copy output to install configuration on nodes
enigma start validator

# FOR EACH NODE `i`
docker run -it --name node<i> -p 7735:7735 -p 7736:7736 enigma:0.1
enigma keygen
echo '<printconfig-output>' > enigma.conf
enigma start node -naddr <node-i-pub-ip>:7735
```

Here we provide an example for setting up an Enigma Network.
We assume that each machine has already installed [Docker](https://github.com)
We assume here that the validator and nodes each run in separate docker containers. 
Once each instance setup, you can use `<CTRL p> + <CTRL q>` to detach from a docker container and leave it running.

<aside class="warning">
Always ensure that nodes connect to the validator in the same order.
The ordering determines the network identifier given to each node.
Failing to do so will prevent shares from being reconstructed properly.
</aside>
