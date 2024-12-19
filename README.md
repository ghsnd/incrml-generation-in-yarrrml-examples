# General overview

This is a general full-fletched overview with all options and describes behaviour of how to generate IncRML with YARRRML.

```yaml
sources:
  data-source-1: a data source
  data-source-2: another data source

targets:
  # an "ordinary" target'
  my-boring-target:
    access: out.nq
    type: localfile
    serialization: nquads
  # an LDES target
  my-special-target:
    access: out-ldes.nq
    type: localfile
    serialization: nquads
    ldes:   # LDES specific keys
      id: https://my-ldes.org/the-one-and-only-ldes  # optional: uri for ldes:EventStream object. Invent a sensible default
      timestampPath: dcterms:created     # optional, default = dcterms:created. Create a corresponding PO in mapping using target if not exists?
      versionOfPath: dcterms:isVersionOf # optional, default = dcterms: isVersionOf. Create a corresponding PO in mapping using target if not exists?
      generateImmutableIRI: false        # optional, default = false. If true, turn the member subject IRI into a unique one.

mapping:
  general-mapping:

    # Here comes some traditional mapping stuff...
    sources: data-source-1
    graphs: ex:some-graph
    subjects:
      - value: ex:$(ObservationID)
        targets: my-boring-target
    po:
      - some-PO-mappings: fun!

    # Here's where the magic happens.
    # Specify what to do when certain changes in data are detected.
    # You can specify any combination of 'create', 'update' and 'delete' here
    changeDetection:

      # The operation to perform when change is detected.
      # Can be `create`, `update` or `delete`.
      # In this case the explicit creation of new data objects
      create:

        # The type of operation: is the create operation explicit or not?
        # Optional.
        #  `true` = explicitly advertised by the data source (default)
        #  `false` = implicitly advertised by the data source
        # See more explanation in section "Change detection".
        explicit: true

        # Optional. Things that will be *added to* the current mapping for this operation
        mappingAdd:
          sources: data-source-2
          graphs: ex:a-second-graph
          subjects:
            - value: ex:$(Sensor)/$(ObservationID)
              targets: my-special-target
          po: [even more fun]

        # Optional. Things that will be removed (ignored) from the original mapping
        mappingRemove:
          subjects: []  # This means "all subjects"
          graphs: []
          po: []
          sources: []

        # Here's an "implicit update" example:
        update:
          explicit: false

          # References to data attributes that trigger an update when they change.
          watchedProperties: [$(temperature)]

          mappingAdd:
            # Add a graph to the original mapping.
            graphs: ex:update

            # Add a target to the original subjects.
            subjects:
              - targets: my-special-target

          # Remove the original graphs at mapping level
          mappingRemove:
            graphs: []

        # The "delete" operation works the same.
```

## Notes, remarks
* The `delete` operation removes all PO maps from the generated triples map.
  By adding POmaps using `mappingAdd`, you can create RDF that provide hints/classifications
  e.g. `ex:id4 ex:currentState <deleted>` . If no `mappingAdd po` elements are added,
  all original `rdf:type` `po`s are kept.

## Examples of explicit / implicit operation spec

```yaml
# explicit create with default options
create:
  explicit: true

# implicit create with default options
create:
  explicit: false

# explicit create with a specific data source
create:
  explicit: true
  mappingRemove:
    sources: []
  mappingAdd:
    sources: create-source

# implicit update with properties to watch for change.
# Results in using the default `implicitUpdate` IDLab function
# to check the properties.
update:
  explicit: false
  watchedProperties: [$(temperature)]

# Custom change detection can be accomplished by just using
# functions at subjects level without specifying changeDetection.
mappings:
  general-mapping:
    subjects:
      - function: idlab-fn:implicitUpdate
        parameters:
          - [idlab-fn:watchedProperty, $(temperature)]
```

# The details

## Change detection

To model changes in data (e.g., modeling a stream of events), we introduce the
`changeDetection` key. It specifies how to detect and act upon changes in
the data of a certain `mapping`.

```yaml

mapping:
  myMapping:
    subjects: subject mappings
    predicateObjects: predicate-object mappings
    graphs: graph mappings

    changeDetection:
      ... # details of the change detection...
```

How changes are detected, depend on the data source: it can publish
changes explicitly or implicitly.
Types of changes are `create`, `update`, and `delete`.
This results in handling the following combinations:

In YARRRML this is defined by an *operation* (create, update, delete) key
and a boolean `explicit` sub-key.

```yaml
changeDetection:
  create:
    explicit: true
```

Implicit create:
```yaml
changeDetection:
  create:
    explicit: false
```

Explicit update:
```yaml
changeDetection:
  update:
    explicit: true
```

Implicit update:
```yaml
changeDetection:
  update:
    explicit: false
```

Explicit delete:
```yaml
changeDetection:
  delete:
    explicit: true
```

Implicit delete:
```yaml
changeDetection:
  delete:
    explicit: false
```

By default, changes are detected by detecting presence or absence of
the IRI generated by the `subjects` mappings.

This translates to one of the IDLab functions
`explicitCreate`, `implicitCreate`, `explicitUpdate`, `implicitUpdate`,
`explicitDelete`, and `implicitDelete` applied to the subject mapping.

The next table illustrates what a subject mapping with change detection generates:

| Run | Incoming IRI  | Create e      | Create i      | Update e      | Update i      | Delete e      | Delete i      |
|-----|---------------|---------------|---------------|---------------|---------------|---------------|---------------|
| 1   | example.org/1 | example.org/1 | example.org/1 | example.org/1 | x             | example.org/1 | x             |
| 2   | example.org/2 | example.org/2 | example.org/2 | example.org/2 | x             | example.org/2 | example.org/1 |
| 3   | example.org/1 | x             | x             | x             | example.org/1 | x             | example.org/2 |

- Every row represents a new 'run', meaning (re-)mapping the data.
  So the first run the dataset consists of `example.org/1`. The second run the dataset consists of `example.org/2`,
  and the third run the dataset consists of `example.org/1` again.
- `e` stands for `explicit`, `i` stands for `implicit`.
- The other cells display the IRI generated by the subject mapping; `x` means the subject mapping is not executed.

**Explicit** and **Implicit create** behave the same: if an IRI is not seen yet, it is considered new and gets
generated by the subject mapping.

**Explicit update** and **Explicit delete** consider the incoming data as updates or deletes resp. and will
get generated by the subject mapping.
Duplicates are ignored: their IRIs *are already* updated or deleted by the data source.

**Implicit update** only considers IRIs that have already been seen as updates, and they get generated by the subject mapping.
If updates in data fields that are not used when generating the subject IRI need to be considered, see *watchedProperties*.

**Implicit delete** considers IRIs it *does not* encounter the *next* run as deleted.
The subject mapping generates those deleted IRIs.

## watchedProperties

Implicit changes sometimes require monitoring certain properties or
attributes of the data that are *not* used for subject IRI generation.

For example, consider this initial dataset:

| sensorID | temperature |
|----------|-------------|
| 1        | 15.1        |
| 2        | 14.9        |

We'd map this in YARRRML as an implicit create because an update would
would use the same sensor IDs.

```yaml
mapping:
  temperatures:
    subjects: https://thermometer.net/sensor_$(sensorID)
    po:
      - [ex:temperature, $(temperature)]
    changeDetection:
      create:
        explicit: false
```

Next we get an update of this data set:

| sensorID | temperature |
|----------|-------------|
| 1        | 17          |
| 2        | 14.9        |

Notice that sensor 1 changes its reading while sensor 2 stays the same.
We want to capture the change in sensor 1's value.

We'd map this in YARRRML as an implicit update:

```yaml
mapping:
  temperatures:
    subjects: https://thermometer.net/sensor_$(sensorID)
    po:
      - [ex:temperature, $(temperature)]
    changeDetection:
      update:
      create:
        explicit: false
      update:
        explicit: false
```
This is not enough though; the result of this mapping would not generate triples:
the `sensorID`s remain the same, so no change is detected in the subject IRI.

To fix this, a `watchedProperties` key can be added: we can specify to
monitor `temperature` for changes:

```yaml
mapping:
  temperatures:
    subjects: https://thermometer.net/sensor_$(sensorID)
    po:
      - [ex:temperature, $(temperature)]
    changeDetection:
      update:
      create:
        explicit: false
      update:
        explicit: false
        watchedProperties: [$(temperature)]
```

When processing the updated dataset the temperature change of sensor 1
will be detected and a new triple will be generated.


## Modifying the original mappings

It is possible to act differently on different changes.
For example, changes could be written to a specific named graph,
or another target.
Or certain `predicateObject` mappings would not be executed for certain
changes.
This chapter descripbes the possibilities.

Modifications in the original mappings can be specified with the
`mappingAdd` and `mappingRemove` sub-keys of `changeDetection`.

### Modifying source mappings

#### Example

Suppose we process a stream of messages like these:

```json
{
  "create": [{"fruit": "apple", "colour": "green"}, {"fruit": "orange", "colour": "orange"}],
  "update": [],
  "delete": []
}
```
```json
{
  "create": [{"fruit": "mellon", "colour": "yellow"}],
  "update": [{"fruit": "apple", "colour": "red"}],
  "delete": []
}
```
```json
{
  "create": [{"fruit": "pear", "colour": "green"}],
  "update": [],
  "delete": [{"fruit": "apple"}, {"fruit": "mellon"}]
}
```

All changes are explicitly advertised in separate JSON keys for every type
of change (what a coincidence!).

One way of specifying this in YARRRML is by removing the
original source(s) and add different sources
since they have a different iterator:

```yaml
# Different iterators, so different sources:
sources:
  create-source: [message.json~jsonpath, $.create.*]
  update-source: [message.json~jsonpath, $.update.*]
  delete-source: [message.json~jsonpath, $.delete.*]

mappings:

  s: http://fruit.org/$(fruit)

  po:
    - [ex:colour, $(colour)]

  changeDetection:
    create:
      explicit: true
      mappingAdd:
        sources: create-source
      mappingRemove:
        sources: []
    update:
      explicit: true
      mappingAdd:
        sources: update-source
      mappingRemove:
        sources: []
    update:
      explicit: true
      mappingAdd:
        sources: delete-source
      mappingRemove:
        sources: []
```

A more concise way achieving the same result
is "updating" the iterator from the source mapping:

```yaml
mappings:

  # This source will be updated by the operations:
  sources: [message.json~jsonpath, $.*]
  s: http://fruit.org/$(fruit)
  po:
    - [ex:colour, $(colour)]

  changeDetection:
    create:
      explicit: true
      mappingAdd:
        sources:
          - iterator: $.create.*
    update:
      explicit: true
      mappingAdd:
        sources:
          - iterator: $.update.*
    create:
      explicit: true
      mappingAdd:
        sources:
          - iterator: $.delete.*
```

#### Possibilities

This is what can be done with `mappingAdd` and `mappingRemove`:

* Add a new source for this operation:
  ```yaml
  changeDetection:
    create:
      mappingAdd:
        # Adds a new source when processing this `create`
        sources: reference-to-a-new-source
  ```
  ```yaml
  changeDetection:
    create:
      mappingAdd:
        # Adds a new source when processing this `create`, this time with
        # an inline source definition
        sources:
          - [data.json~jsonpath, $.*]
  ```
* Change the original source for this operator can be done by omitting
  the `access` key at `mappingAdd/sources` level:
  ```yaml
  changeDetection:
    create:
      mappingAdd:
        # Updates the original source with a new iterator:
        sources:
          - iterator: $.*
  ```
  The same can be done for all other sub-keys of `sources`, such as
  `delimiter`, `query`, `encoding`, etc.
* Remove all sources:
  ```yaml
  changeDetection:
    create:
      mappingRemove:
        sources: []
  ```
* Remove certain references to sources:
  ```yaml
  changeDetection:
    create:
      mappingRemove:
        sources:
          - reference-to-a-source
  ```
* Remove certain keys from the original sources
(in this case `compression` and `delimiter`):
  ```yaml
  changeDetection:
    create:
      mappingRemove:
        sources:
          - compression
          - delimiter
  ```
* Remove certain keys from the original sources and all graphs:
(in this case `compression` and `delimiter`):
  ```yaml
  changeDetection:
    create:
      mappingRemove:
        sources:
          - compression
          - delimiter
        graphs:
  ```
* Remove all predicate-object mappings:
  ```yaml
  changeDetection:
    create:
      mappingRemove:
        predicateobjects: []
  ```
  or
  ```yaml
  changeDetection:
    create:
      mappingRemove:
        po: []
  ```
* Remove a specific predicate-object mapping:
  ```yaml
  changeDetection:
    create:
      mappingRemove:
        po: [["ex:pressure", "$(pressure)"]]
  ```
* Remove a specific subject mapping:
  ```yaml
  changeDetection:
    create:
      mappingRemove:
        subjects: ex:sensor/$(sensor)
  ```
* Remove all targets from all subjects:
  ```yaml
  changeDetection:
    create:
      mappingRemove:
        subjects:
          targets: []
  ```
* Remove all targets from all subjects *and* remove a specific subject
  ```yaml
  changeDetection:
    create:
      mappingRemove:
        subjects:
          - ex:sensor/$(id)  # specific subject
          - targets: []      # all targets
  ```

**Not possible yet**:
* Remove all po mappings with a given predicate or a given object.
* Remove/add graphs from/to PO mappings.
* Remove/add data types from/to PO mappings.
* Add/remove inverse predicates
* Remove sub-keys of targets, e.g. `compression`.
