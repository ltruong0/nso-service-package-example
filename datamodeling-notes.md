# Data Modeling
#cisco_nso

Fundamental Elements (node types) for describing data:
- leaf nodes
- leaf-list nodes
- container nodes
- list nodes

Yang separates nodes into those that hold data (leaf, leaf-list) and those that hold other nodes (container, list)

A `leaf` contains simple data such as integer or string. One value of a particular type and no child nodes

example
```
leaf host-name {
	type string;
	description "Hostname for this system";
}
```
In nso setting a host-name would look like
```
admin@ncs(config)# host-name "server-NY-01"
```

`leaf-list` is a sequence of leaf nodes of the same type. It can hold multiple values like an array.
```
leaf-list domains {
	type string;
	description "My favorite internet domains";
}
```
In NSO CLI assigning multiple values to a leaf-list can be done using square bracket syntax:
```
admin@ncs(config)# domains [ cisco.com tail-f.com ]
```

A `container` node is used to group related nodes into a subtree. 
> As a model keeps expanding, having all data nodes on the same (top) level can quickly become unwieldy. 
> It has only child nodes and no value. 
> Container may contain any number of child nodes of any type (including leafs, lists, containers, and leaf-lists).
example
```
container server-admin {
    description "Administrator contact for this system";
    leaf name {
        type string;
    }
}
```
In CLI :
```
admin@ncs(config)# server-admin name "Ingrid"
```

A `list` defines a collection of container-like entries that share the same structure.
> Each entry is similar to a record or row in a table
> Uniquely identified by value of its key leaf (or leaves)
> Can container any number of child nodes of any type (leafs, container, other lists, and so on)
Example
```
list user-info {
    description "Information about team members";
    key "name";
    leaf name {
        type string;
    }
    leaf expertise {
        type string;
    }
}
```

In CLI lists take an additional parameter, the key value, in order to select a single entry

```
admin@ncs(config)# user-info "Ingrid"
```

To set a value of a particular list entry, specific the entry and then the child node
```
admin@ncs(config)# user-info "Ingrid" expertise "Linux"
```