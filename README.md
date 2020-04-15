# End User Definitions Documentation

The
[docs for the LifeCycle Manager API that serves definitions](https://docs.datastax.com/en/opscenter/6.1/api/docs/lcm_definitions.html#lcm-definitions)
provide additional information about some of the definitions.

# Definitions Concepts

## File Formats

Definitions make use of both EDN (Extensible Data Notation) formatted files.

See: [The Extensible Data Notation Reference](https://github.com/edn-format/edn)

# Config DDL Reference

The data definition language for config files provides a schema for configs
that allows understanding all Cassandra and DSE configs as maps/dictionaries, even when
the config itself is unstructured data like cassandra-env.sh. Config DDL files
are found in directories like
[resources/cassandra-yaml/](https://github.com/datastax/cass-config-definitions/tree/master/resources/cassandra-yaml/dse).

## Base Files

Base files define a complete description of a config file, for example the
[cassandra.yaml for DSE 5.1.0](https://github.com/datastax/cass-config-definitions/blob/master/resources/cassandra-yaml/dse/cassandra-yaml-dse-5.1.0.edn).

Note: Certain fields and attributes are only useful in automatic rendering of a User Interface, and are not documented.

### Simplified Example Base File

``` edn
{:display-name "file name"
 :workload-file-group "workload name"
 :ui-visibility :editable
 :renderer {:renderer-type :yaml}
 :properties {:my-checkbox {:type "boolean"
                            :default_value true
                            :description "A boolean configuration option"}
              :my-textbox {:type "string"
                           :default_value "A string!"
                           :depends :my-checkbox
                           :description "A string that is only present when the checkbox is ticked."}
              :my-foo     {:type "string"
                           :depends :my-textbox
                           :conditional [{:eq "A string!"}]
                           :description "A field that has a transitive dependency on :my-checkbox."}
              }
 :groupings [{:name "My Group",
              :description "Both settings will be grouped under a 'My Group' heading in the UI",
              :list ["my-checkbox" "my-textbox"]}]}
```

### Base File Attributes

- *:renderer* Specifies how to render the template, accepts a map of either
  `{:renderer-type :yaml}` or
  `{:renderer-type :template, :template "filename.template"}`
- *:properties* A nested dictionary containing field-metadata. Each top-level
  key is the name of a field, each value is a dictionary containing field
  metadata attributes describing the field. See the field metadata section
  for details.

### Field Metadata

Field metadata is defined as a map of keys and values within the top-level
'properties' key. Each key defines a field-name, and each value is a dictionary
describing the field.

- *:type* An enum of boolean, int, string, list, or dict.
- *:default_value* The default value that will be supplied.
- *:description* A human-readable description.
- *:unit* Currently unused, could be used to display a string
  indicating what unit a config is in (megabytes, milliseconds, etc).
- *:depends* Some fields must be included/excluded based on the state of
  another field. For example, it makes no sense to set the node-to-node keystore
  file if node-to-node encryption is disabled. The value of this attribute
  specifies the name of another field whose state controls whether this field
  is rendered.
- *:conditional* When the parent field in a depends relationship is a boolean,
  the condition of dependency is obvious and may remain implicit. In cases where
  the parent field is a string or an enum, it is sometimes necessary to define
  the condition explicitly. For example
  `:conditional [{:eq "my-val"} {:eq "other-val"}]` would
  allow a child to depend on the value of a parent string to be equal to
  "my-val" or "other-val".
- *:value_type* Fields of type 'list' must specify a value_type for list-members
  which is an enum that accepts the same options as the 'type' attribute.
- *:render_as* Some config-options are represented in config-files as 0 or 1
  when they are conceptually booleans. This allows us to present a boolean
  api and render a 0 or 1 in the output: `:render_as "int"`
- *:options* Fields of type 'list' can be converted to a dropdown/enum by adding
  a list of valid options like
  `:options [{:label "human readable name for value", :value "config-value"}]`
- *:fields* Fields of type 'dict' can contain subfields, which are defined as
  nested maps under this key, for example:
  ``` edn
  :my_dict_field {:type "dict"
                  :description "A dictionary field containing sub-fields"
                  :fields {:sub_field_1 {:type "string"
                                         :default_value "my val"
                                         :description "A string member"}
                           :sub_field_2 {:type "int"
                                         :default_value 1
                                         :description "An int member"}}}
  ```
- *:constant* Some fields are rendered into bash files as variables or exports.
  When a value for 'constant' is declared, that becomes the key-name for the
  bash variable or export. If you want the output `MY_VAR="val"` you would use
  a field definition like:
  `:my_bash_var {:type "string", :constant "MY_VAR", :default_value "val"}`
- *:add-export* When used in conjunction with 'constant', indicates that the
  rendered result should look like a variable export in bash. If you want the
  output `export MY_VAR="val"` you would use a field definition like:
  `:my_bash_var {:type "string", :constant "MY_VAR", :add-export true, :default_value "val"}`
- *:render-without-quotes* When used in conjunction with 'constant', renders
  values unquoted. If you want the output `MY_VAL=val` you would use a
  field definition like:
  `:my_bash_var {:type "string", :constant "MY_VAR", :render-without-quotes true, :default_value "val"}`
- *:supress-equal-sign* When used in conjunction with 'constant', renders the
  constant's key and value adjacent to each other with no '=' between them.
  If you want to render `-Xmx2048`, you would use a field definition like:
  `:my_jvm_opt {:type "string", :constant "-Xmx", :suppress-equal-sign true, :render-without-quotes true}`
- *:exclude-from-template-iterables* For an example, jvm.options template iterates through
  a list of fields rendering each one using identical logic. If a field needs to
  opt out of this logic it must be tagged with
  `:exclude-from-template-iterables true`. This attribute has no effect in
  templates or config files that do not iterate over a list of variables
  the way jvm.options does.
- *:static_constant* Similar to 'constant' fields, but applies to booleans where
  the value simply controls whether the content is rendered at all. If you want
  to conditionally render `-XX:+HeapDumpOnOutOfMemoryError` or nothing at all,
  depending on the value of a boolean checkbox, you would use a field definition
  like: `:heap_dump_on_out_of_memory_error {:type "boolean", :static_constant "-XX:+HeapDumpOnOutOfMemoryError"}`
- *:is_directory* A truthy value indicates that this field represents a directory
  or list of directories.
- *:is_file* A truthy value indicates that this field represents a file

### User Defined Values

It's possible to create dictionary fields that contain members where the keys,
not just the values, are user-defined. This is not currently documented, but
there are examples in the definitions, grep for 'user_defined'.

## Transforms

Transforms allow us to specify a series of changes to a basefile over time.

Basefiles are a useful way of specifying a complete set of definitions for a
version of DSE or Cassandra, but they are extremely verbose. If definitions were 
expressed for every version as a basefile, correcting a mistake that affects
definitions for 6 patch releases within a stable series would be extremely
repetitive and error prone. Transforms allow the ability to efficiently express
changes relative to a previous state.

### Simplified Example Transforms File

This example covers DSE 5.1.0-5.1.5.

- The definitions for DSE 5.1.0-5.1.2 are identical in this example.
- DSE 5.1.3 contains all the definitions from DSE 5.1.2, and also introduces
  the new 'new-field' definition.
- DSE 5.1.4 contains all the definition from DSE 5.1.3, including 'new-field',
  and further adds 'another-new-field'.
- DSE 5.1.5 contains all the definitions from DSE 5.1.4, including both new
  fields.
- DSE 6.0.0 starts with a new basefile, and shares nothing with the 5.1.x
  series. Future 6.0.x transforms will be made relative to that new basefile.

``` edn
("5.1.0" "dse-default-dse-5.0.1.edn"
 "5.1.3" (transforms
          (add-field :new-field
                     {:type "int"
                      :default_value 14
                      :unit "seconds"}
                     :group "General"))
 "5.1.4" (transforms
          (add-field :another-new-field
                     {:type "int"
                      :default_value 15
                      :unit "seconds"}
                     :group "General"))
 "6.0.0" "dse-default-dse-6.0.0.edn)
```

### Available Transforms

Transforms are expressed as an EDN list where odd members are DSE versions and
even members are a valid transform. The available transforms are defined with
helpful docstrings in lcm.config.generator, and are summarized below for
convenience:

- *string containing a basefile* A string is assumed to contain the filename
  for a basefile in the same directory as the transforms file. When a new
  basefile is introduced, it must specify a complete set of definitions. No
  content from previous transforms is carried over in this case.
- *transforms expression* An expression that can contain several potential
  transformations.
- *add-field* When inside a 'transforms' expression, adds a new field. Contains
  a field definition as it would appear in a basefile, plus an additional
  `:group` key that specifies a pre-existing group to which the new field
  is being added. Note that only top-level fields need a `:group` key,
  sub-members of lists and dictionaries implicitly belong to the group
  of the top-level list or dictionary. See the transforms example for an
  example of how this is used.
- *update-field* When inside a 'transforms' expression, allows modifying
  attributes of an existing field. To change the default value of new-field
  in DSE 5.1.5, one would do the following:
  `"5.1.5" (transforms (update-field :new-field {:default-value 25}))`
- *delete-field* When inside a 'transforms' expression, removes a field:
  `"5.1.5" (transforms (delete-field :new-field))`
- *add-group* When inside a 'transforms' expression, adds a new group.
- *delete-group* When inside a 'transforms' expression, adds a new group.
- *update-template* When inside a 'transforms' expression, transforming a
  basefile that has a renderer-type of template, one can update the name of
  the selmer template file in the same directory as the transforms that is
  used to render the config output via:
  `"5.1.5 (transforms (update-template "new.template"))`.

### Dot Separated Paths

Dictionaries can can have members updated, added, and removed by
referencing members via a dot-separated path. When updating large dictionaries
this can save a large amount of error-prone duplication.

For example to update the my-bool member of my-dict:

``` edn
 ("5.1.0" "dse-default-dse-5.0.1.edn"

 ;; Adds a dictionary my-dict with a member my-bool
 "5.1.1" (transforms
          (add-field :my-dict
                     {:type "dict"
                      :fields {:my-bool {:type "boolean"
                                         :default_value true}}}
                     :group "General"))

 ;; Changes the default value of my-bool using a dot-separated path
 "5.1.2" (transforms
          (update-field :my-dict.my-bool {:default_value false})))
```
