nagios-plugin-check_galera_cluster
=======================================

A nagios plugin to check status of a galera cluster

    Version 1.1, Guillaume Coré <fridim@onfi.re>, Ales Nosek <ales.nosek@gmail.com>

    check_galera_cluster is a Nagios plugin to monitor Galera cluster status.
    
    check_galera_cluster [-u USER] [-p PASSWORD] [-H HOST] [-P PORT] [-w SIZE] [-c SIZE] [-f FLOAT] [-0]
    
    Options:
      u)
        MySQL user.
      p)
        MySQL password.
      H)
        MySQL host.
      P)
        MySQL port.
      w)
        Sets minimum number of nodes in the cluster when WARNING is raised. (default is same as critical).
      c)
        Sets minimum number of nodes in the cluster when CRITICAL is raised. (default is 2).
      f)
        Sets critical value of wsrep_flow_control_paused (default is 0.1).
      0)
        Rise CRITICAL if the node is not primary
