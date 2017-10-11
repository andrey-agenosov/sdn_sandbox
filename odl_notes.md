## Interaction with ODL via REST API

* Get **current** inventory: *curl -u <LOGIN> -v http://<CONTROLLER_IP>:<CONTROLLER_PORT>/restconf/operational/opendaylight-inventory:nodes*
* Get **current** topology: *curl -u <LOGIN> http://<CONTROLLER_IP>:<CONTROLLER_PORT>/restconf/operational/network-topology:network-topology/topology/flow:1*

### Working with Brocade's PySdn

* Launch the ODL controller and Mininet to connect switch to the ODL: *sudo mn --topo=tree,1,2 --controller=remote,127.0.0.1:6653*
* View network topology with details about switches and hosts: *oftool -C ./ctrl.yml show-topo -v*