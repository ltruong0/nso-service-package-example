# NSO Service Package Example
---
> Showcase example of creating a Service Package


## Setup
---
> This setup assumes Cisco NSO is already installed and setup on the host machine

`Create Package` :
```
# CD into package directory (ex: nso-instance/packages)
cd nso-instance/packages

# create a new package
ncs-make-package --service-skeleton template <service_name>

# The example here is `loopback-service`
ncs-make-package --service-skeleton template loopback-service
```
---
`src/yang/loopback-service.yang` :
> This is created with the `ncs-make-package` command, and defines the variables that will be used for configuration
> of the service.

```
module loopback-service {
  namespace "http://com/example/loopbackservice";
  prefix loopback-service;

  import ietf-inet-types {
    prefix inet;
  }
  import tailf-ncs {
    prefix ncs;
  }

  list loopback-service {
    key name;

    uses ncs:service-data;
    ncs:servicepoint "loopback-service";

    leaf name {
      type string;
    }

    // may replace this with other ways of refering to the devices.
    leaf-list device {
      type leafref {
        path "/ncs:devices/ncs:device/ncs:name";
      }
    }

    // replace with your own stuff here
    leaf dummy {
      type inet:ipv4-address;
    }
  }
}

```
- The YANG file has a list (list loopback-service) to track all the service instances (more on this topic later).
- Within the service definition, there are a few leaf variable names, including name, device, and dummy. A leaf is simple a single key:value input.
- The name variable is the service instance unique key lookup (more on that topic later too).
- The device variable is a dynamic reference in the NSO CDB to the device list.
- The dummy variable is a sample input to use in a device template, with a variable type of IP address.
---
`loopback-service/templates/loopback-service-template.xml`

```
<config-template xmlns="http://tail-f.com/ns/config/1.0"
                 servicepoint="loopback-service">
  <devices xmlns="http://tail-f.com/ns/ncs">
    <device>
      <!--
          Select the devices from some data structure in the service
          model. In this skeleton the devices are specified in a leaf-list.
          Select all devices in that leaf-list:
      -->
      <name>{/device}</name>
      <config>
        <!--
            Add device-specific parameters here.
            In this skeleton the service has a leaf "dummy"; use that
            to set something on the device e.g.:
            <ip-address-on-device>{/dummy}</ip-address-on-device>
        -->
      </config>
    </device>
  </devices>
</config-template>

```
- The XML file is the configuration NSO uses to apply to your network devices. You plug in the variable names from your YANG file into here to have dynamic configuration. You can also use multiple - YANG files (more often with extra Python or Java code logic).
- The important part of the XML file which you use is between the tags <config> and </config>, which right now has an XML comment.

## Update/Modify Template
---
> To easily gather XML configurations to plug into our template,
> you can view the configuration from the device itself

`Access the ncs cli`
```
ncs_cli -C -u admin
```

`Configure a device with loopback`
```
devices device cisco-dev config
interface Loopback 100
ip address 10.10.30.0 255.255.255.0
```

`View the configuration (no need to actually commit here)`
```
show configuration
commit dry-run outformat xml
```

`Example XML output`
```
result-xml {
    local-node {
        data <devices xmlns="http://tail-f.com/ns/ncs">
               <device>
                 <name>cisco-dev</name>
                 <config>  <--------------------------
                   <interface xmlns="urn:ios"> 
                     <Loopback>
                       <name>100</name>
                       <ip>
                         <address>
                           <primary>
                             <address>10.10.30.0</address>
                             <mask>255.255.255.0</mask>
                           </primary>
                         </address>
                       </ip>
                     </Loopback>
                   </interface>
                 </config>  <--------------------------
               </device>
             </devices>
    }
}

```
- The important portion here is between the `config` tags, the snippet in between will be used within the template.


Update `loopback-service/templates/loopback-service-template.xml` with configuration
```
<config-template xmlns="http://tail-f.com/ns/config/1.0"
                 servicepoint="loopback-service">
  <devices xmlns="http://tail-f.com/ns/ncs">
    <device>
      <!--
          Select the devices from some data structure in the service
          model. In this skeleton the devices are specified in a leaf-list.
          Select all devices in that leaf-list:
      -->
      <name>{/device}</name>
       <config>
                    <interface xmlns="urn:ios">
                      <Loopback>
                        <name>100</name>
                        <ip>
                          <address>
                            <primary>
                              <address>{/dummy}</address>
                              <mask>255.255.255.0</mask>
                            </primary>
                          </address>
                        </ip>
                      </Loopback>
                    </interface>
       </config>
    </device>
  </devices>
</config-template>
```
- Notice that the configuration has been replaced with the XML snippet.
- `{/dummy}` variable replaces the static address

`Compile Yang Model`

```
cd loopback-service/src/
make
```

`Reload Package`
```
ncs_cli -C -u admin
packages reload
```

## Instantiation
---
> This section will go over creating a service instance using the template/model created above

```
# enter config mode
config

# configure service
<service_name> <instance_name>
device <device>
dummy <value>


# Example
admin@ncs# config
Entering configuration mode terminal
admin@ncs(config)# loopback-service test
admin@ncs(config-loopback-service-test)# device cisco-dev
admin@ncs(config-loopback-service-test)# dummy 192.168.1.1

# view the config
admin@ncs(config-loopback-service-test)# top

# native output
admin@ncs(config)# commit dry-run outformat native
native {
    device {
        name cisco-dev
        data interface Loopback100
              ip address 192.168.1.1 255.255.255.0
              no shutdown
             exit
    }
}

# view config changes
admin@ncs(config)# show configuration
loopback-service test
 device [ cisco-dev ]
 dummy  192.168.1.1
!

# commit the changes
admin@ncs(config)# commit
Commit complete.
admin@ncs(config)#
```