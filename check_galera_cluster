#!/bin/bash

LANG=C

PROGNAME=`basename $0`
VERSION="Version 1.1.5"
AUTHOR="Guillaume Coré <fridim@onfi.re>, Ales Nosek <ales.nosek@gmail.com>, Staf Wagemakers <staf@wagemakers.be>, Claudio Kuenzler <claudiokuenzler.com>"

ST_OK=0
ST_WR=1
ST_CR=2
ST_UK=3

warnAlerts=0
critAlerts=0
unknAlerts=0


print_version() {
  echo "$VERSION $AUTHOR"
}

print_help() {
  print_version $PROGNAME $VERSION
  echo ""
  echo "$PROGNAME is a monitoring plugin to monitor Galera cluster status."
  echo ""
  echo "$PROGNAME [-u USER] [-p PASSWORD] [-H HOST] [-P PORT] [-m file] [-w SIZE] [-c SIZE] [-s statefile] [-f FLOAT] [-0]"
  echo ""
  echo "Options:"
  echo "  u)"
  echo "    MySQL user."
  echo "  p)"
  echo "    MySQL password."
  echo "  H)"
  echo "    MySQL host."
  echo "  P)"
  echo "    MySQL port."
  echo "  m)"
  echo "    MySQL extra my.cnf configuration file."
  echo "  w)"
  echo "    Sets minimum number of nodes in the cluster when WARNING is raised. (default is same as critical)."
  echo "  c)"
  echo "    Sets minimum number of nodes in the cluster when CRITICAL is raised. (default is 2)."
  echo "  f)"
  echo "    Sets critical value of wsrep_flow_control_paused (default is 0.1)."
  echo "  0)"
  echo "    Rise CRITICAL if the node is not primary"
  echo "  s)"
  echo "    Create state file, detect disconnected nodes"
  exit $ST_UK
}

# default values
crit=2
fcp=0.1

check_executable() {
    if [ -z "$1" ]; then
        echo "check_executable: no parameter given!"
        exit $ST_UK
    fi

    if ! command -v "$1" &>/dev/null; then
        echo "UNKNOWN: Cannot find $1"
        exit $ST_UK
    fi
}

check_executable mysql
check_executable bc

while getopts “hvu:p:H:P:w:c:f:m:s:0” OPTION; do
  case $OPTION in
    h)
      print_help
      exit $ST_UK
      ;;
    v)
      print_version $PROGNAME $VERSION
      exit $ST_UK
      ;;
    u)
      mysqluser=$OPTARG
      ;;
    p)
      password=$OPTARG
      ;;
    H)
      mysqlhost=$OPTARG
      ;;
    P)
      port=$OPTARG
      ;;
    m)
      myconfig=$OPTARG
      ;;
    w)
      warn=$OPTARG
      ;;
    c)
      crit=$OPTARG
      ;;
    f)
      fcp=$OPTARG
      ;;
    0)
      primary='TRUE'
      ;;
    s)
      stateFile=$OPTARG
      ;;
    ?)
      echo "Unknown argument: $1"
      print_help
      exit $ST_UK
      ;;
  esac
done

if [ -z "$warn" ]; then
  warn=$crit
fi

create_param() {
  if [ -n "$2" ]; then
    echo $1$2
  fi
}

param_mysqlhost=$(create_param -h "$mysqlhost")
param_port=$(create_param -P "$port")
param_mysqluser=$(create_param -u "$mysqluser")
param_password=$(create_param -p "$password")
param_configfile=$(create_param --defaults-extra-file= "$myconfig")
export MYSQL_PWD=$password

param_mysql="$param_mysqlhost $param_port $param_mysqluser $param_configfile"

#
# verify the database connection
#

mysql $param_mysql -B -N  -e '\s;' >/dev/null 2>&1 || {
  echo "CRITICAL: mysql connection check failed"
  exit $ST_CR
}

#
# retrieve the mysql status
#

rMysqlStatus=$(mysql $param_mysql -B -N -e "show status like 'wsrep_%';")

#
# verify that the node is part of a cluster
#

rClusterStateUuid=$(echo "$rMysqlStatus" | awk '/wsrep_cluster_state_uuid/ {print $2}')

if [ -z "$rClusterStateUuid" ]; then
  echo "CRITICAL: node is not part of a cluster"
  exit $ST_CR
fi

rClusterSize=$(echo "$rMysqlStatus" | awk '/wsrep_cluster_size/ {print $2}')
rClusterStatus=$(echo "$rMysqlStatus" | awk '/wsrep_cluster_status/ {print $2}') # Primary
rFlowControl=$(echo "$rMysqlStatus" | awk '/wsrep_flow_control_paused\t/ {print $2}') # < 0.1
rFlowControl=$(printf "%.14f" $rFlowControl) # issue #4
rReady=$(echo "$rMysqlStatus" | awk '/wsrep_ready/ {print $2}') # ON
rConnected=$(echo "$rMysqlStatus" | awk '/wsrep_connected/ {print $2}') # ON
rLocalStateComment=$(echo "$rMysqlStatus" | awk '/wsrep_local_state_comment/ {print $2}') # Synced
rIncommingAddresses=$(echo "$rMysqlStatus" | awk '/wsrep_incoming_addresses/ {print $2}') 
  
if [ -z "$rFlowControl" ]; then
  echo "UNKNOWN: wsrep_flow_control_paused is empty"
  unknAlerts=$(($unknAlerts+1))
fi

if [ $(echo "$rFlowControl > $fcp" | bc) = 1 ]; then
  echo "CRITICAL: wsrep_flow_control_paused is > $fcp"
  critAlerts=$(($criticalAlerts+1))
fi

if [ "$primary" = 'TRUE' ]; then
  if [ "$rClusterStatus" != 'Primary' ]; then
    echo "CRITICAL: node is not primary (wsrep_cluster_status)"
    critAlerts=$(($criticalAlerts+1))
  fi
fi

if [ "$rReady" != 'ON' ]; then
  echo "CRITICAL: node is not ready (wsrep_ready)"
  critAlerts=$(($criticalAlerts+1))
fi

if [ "$rConnected" != 'ON' ]; then
  echo "CRITICAL: node is not connected (wsrep_connected)"
  critAlerts=$(($criticalAlerts+1))
fi

if [ "$rLocalStateComment" != 'Synced' ]; then
   echo "CRITICAL: node is not synced - actual state is: $rLocalStateComment (wsrep_local_state_comment)"
   critAlerts=$(($criticalAlerts+1))
fi

if [ $rClusterSize -gt $warn ]; then
  # only display the ok message if the state check not enabled
  if [ -z "$stateFile" ]; then
    echo "OK: number of NODES = $rClusterSize (wsrep_cluster_size)"
  fi
elif [ $rClusterSize  -le $crit ]; then
  echo "CRITICAL: number of NODES = $rClusterSize (wsrep_cluster_size)"
  critAlerts=$(($criticalAlerts+1))
elif [ $rClusterSize -le $warn ]; then
    echo "WARNING: number of NODES = $rClusterSize (wsrep_cluster_size)"
    warnAlerts=$(($warnAlerts+1))
  else
   exit $ST_UK
fi

#
# detect is the connection is lost automatically
#

if [ ! -z "$stateFile" ]; then

  touch $stateFile

  if [ $? != "0" ]; then

    echo "UNKNOWN: stateFile \"$stateFile\" is not writeable"
    unknAlerts=$(($unknAlerts+1))

  else

    if [ "$rConnected" = "ON" ]; then
      # get the current connected Nodes
      currentNodes=$(echo $rIncommingAddresses | tr "," "\n" | sort -u)
      if [ -f "$stateFile" ]; then
        # get the nodes added to the cluster
        newNodes=$(echo $currentNodes | tr " " "\n" | comm -2 -3 - $stateFile)
        # get the nodes that were removed from the cluster
        missingNodes=$(echo $currentNodes | tr " " "\n" | comm -1 -3 - $stateFile)
        if [ ! -z "$newNodes" ]; then
          # add the new nodes to the cluster to the state file
          echo $newNodes | tr " " "\n" >> $stateFile
        fi
      else
        # there is no state file yet, creating new one.
        echo $currentNodes | tr " " "\n" > $stateFile
      fi # -f stateFile
      # get the numeber of nodes that were part of the cluster before
      maxClusterSize=$(cat $stateFile | wc -l)

      if [ $maxClusterSize -eq  $rClusterSize ]; then
        if [ $maxClusterSize -eq 1 ]; then
            if [ $crit -eq 0 -a  $warn -eq 0 ]; then
              echo "OK: running single-node database cluster"
            fi
        else
            echo "OK: running redundant $rClusterSize online / $maxClusterSize total"
        fi
      else
            echo "WARNING: redundant  $rClusterSize online / $maxClusterSize  total, missing peers: $missingNodes" 
            warnAlerts=$(($warnAlerts+1))
      fi
  
    fi # rConnected

  fi # -w stateFile

fi # -z stateFile


#
# exit
#

[ "$critAlerts" -gt "0" ] && exit $ST_CR
[ "$unknAlerts" -gt "0" ] && exit $ST_UK
[ "$warnAlerts" -gt "0" ] && exit $ST_WR

exit 0
