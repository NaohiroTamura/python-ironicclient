===================================================
Creating the Bare Metal service resources from file
===================================================

It is possible to create a set of resources using their descriptions in JSON
or YAML format. It can be done in one of three ways:

1. Using ironic CLI's ``ironic create`` command::

    $ ironic help create
    usage: ironic create <file> [<file> ...]

    Create baremetal resources (chassis, nodes, and ports). The resources may
    be described in one or more JSON or YAML files. If any file cannot be
    validated, no resources are created. An attempt is made to create all the
    resources; those that could not be created are skipped (with a
    corresponding error message).

    Positional arguments:
      <file>  File (.yaml or .json) containing descriptions of the resources
              to create. Can be specified multiple times.

2. Using openstackclient plugin command ``openstack baremetal create``::

    $ openstack -h baremetal create
    usage: openstack [-h] [-f {json,shell,table,value,yaml}] [-c COLUMN]
                     [--max-width <integer>] [--noindent] [--prefix PREFIX]
                     [--chassis-uuid <chassis>] [--driver-info <key=value>]
                     [--property <key=value>] [--extra <key=value>]
                     [--uuid <uuid>] [--name <name>]
                     [--network-interface <network_interface>]
                     [--resource-class <resource_class>] [--driver <driver>]
                     [<file> [<file> ...]]

    Create resources from files or Register a new node (DEPRECATED). Create
    resources from files (by only specifying the files) or register a new
    node by specifying one or more optional arguments (DEPRECATED, use
    'openstack baremetal node create' instead).

    positional arguments:
      <file>                File (.yaml or .json) containing descriptions of
                            the resources to create. Can be specified
                            multiple times. If you want to create resources,
                            only specify the files. Do not specify any of
                            the optional arguments.

   .. note::
       If the ``--driver`` argument is passed in, the behaviour of the command
       is the same as ``openstack baremetal node create``, and positional
       arguments are ignored. If it is not provided, the command does resource
       creation from file(s), and only positional arguments will be taken into
       account.

3. Programmatically using the Python API:

    .. autofunction:: ironicclient.v1.create_resources.create_resources
       :noindex:

File containing Resource Descriptions
=====================================

The resources to be created can be described either in JSON or YAML. A file
ending with ``.json`` is assumed to contain valid JSON, and a file ending with
``.yaml`` is assumed to contain valid YAML. Specifying a file with any other
extension leads to an error.

The resources that can be created are chassis, nodes, and ports. Only chassis
and nodes are accepted at the top level of the file structure, but chassis and
nodes themselves can contain nodes or ports definitions nested under ``nodes``
(in case of chassis) or ``ports`` (in case of nodes) keys.

The schema used to validate the supplied data is the following::

    {
        "$schema": "http://json-schema.org/draft-04/schema#",
        "description": "Schema for ironic resources file",
        "type": "object",
        "properties": {
            "chassis": {
                "type": "array",
                "items": {
                    "type": "object"
                }
            },
            "nodes": {
                "type": "array",
                "items": {
                    "type": "object"
                }
            }
        },
        "additionalProperties": False
    }

More detailed description of the creation process can be seen in the following
sections.

Examples
========

Here is an example of the JSON file that can be passed to the ``create``
command::

    {
        "chassis": [
            {
                "description": "chassis 3 in row 23",
                "nodes": [
                    {
                        "name": "node-3",
                        "driver": "agent_ipmitool",
                        "ports": [
                            {
                                "address": "00:00:00:00:00:02"
                            },
                            {
                                "address": "00:00:00:00:00:03"
                            }
                        ]
                    },
                    {
                        "name": "node-4",
                        "driver": "agent_ipmitool",
                        "ports": [
                            {
                                "address": "00:00:00:00:00:04"
                            },
                            {
                                "address": "00:00:00:00:00:01"
                            }
                        ]
                    }
                ]
            }
        ],
        "nodes": [
            {
                "name": "node-5",
                "driver": "pxe_ipmitool",
                "chassis_uuid": "74d93e6e-7384-4994-a614-fd7b399b0785",
                "ports": [
                    {
                        "address": "00:00:00:00:00:00"
                    }
                ]
            },
            {
                "name": "node-6",
                "driver": "pxe_ipmitool"
            }
        ]
    }

Creation Process
================

#. The client deserializes the files' contents and validates that the top-level
   dictionary in each of them contains only "chassis" and/or "nodes" keys,
   and their values are lists. The creation process is aborted if any failure
   is encountered in this stage. The rest of the validation is done by the
   ironic-api service.

#. Each resource is created via issuing a POST request (with the resource's
   dictionary representation in the body) to the ironic-api service. In the
   case of nested resources (``"nodes"`` key inside chassis, or ``"ports"``
   key inside nodes), the top-level resource is created first, followed by the
   sub-resources. For example, if a chassis contains a list of nodes, the
   chassis will be created first followed by the creation of each node. The
   same is true for ports described within nodes.

#. If a resource could not be created, it does not stop the entire process.
   Any sub-resources of the failed resource will not be created, but otherwise,
   the rest of the resources will be created if possible. Any failed resources
   will be mentioned in the response.
