nagios-plugin-check_galera_cluster
=======================================

A nagios plugin to check status of a galera cluster
Version 1.0, Guillaume Coré <g@fridim.org>

    check_galera_cluster is a Nagios plugin to monitor Galera cluster status.
    
    check_galera_cluster -u USER -p PASSWORD [-H HOST] [-P PORT] [-w SIZE] [-c SIZE] [-0]
    
    Options:
      u)
        MySQL user.
      p)
        MySQL password.
      H)
        MySQL host. Default is localhost.
      P)
        MySQL port. Default is 3306.
      w)
        Sets minimum number of nodes in the cluster when WARNING is raised. (default is same as critical).
      c)
        Sets minimum number of nodes in the cluster when CRITICAL is raised. (default is 2).
      0)
        Rise CRITICAL if the node is not primary
