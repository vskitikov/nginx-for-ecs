# nginx-for-ecs

###After download files to storege

    chmod -R 65534:65534 /path/files/

###Code for CloudFormation

        "Fn::Base64": !Join
          -
            ""
          -
            - !If
              - SetEndpointToECSAgent
              - !Sub |
                 #!/bin/bash
                 echo ECS_CLUSTER=${EcsClusterName} >> /etc/ecs/ecs.config
                 echo ECS_BACKEND_HOST=${EcsEndpoint} >> /etc/ecs/ecs.config
              - !Sub |
                 #!/bin/bash
                 echo ECS_CLUSTER=${EcsClusterName} >> /etc/ecs/ecs.config
            - |
               yum update -y
               yum install -y jq nfs-utils python27 python27-pip awscli
               pip install --upgrade awscli
               EFS_FILE_SYSTEM_NAME="NEME-EFS"
               EC2_AVAIL_ZONE="$(curl -s http://169.254.169.254/latest/meta-data/placement/availability-zone)"
               EC2_REGION="$(curl -s http://169.254.169.254/latest/dynamic/instance-identity/document | jq -r '.region')"
               DIR_TGT="/mnt/efs"
               mkdir "${DIR_TGT}"
               EFS_FILE_SYSTEM_ID="$(/usr/local/bin/aws efs describe-file-systems --region "${EC2_REGION}" | jq '.FileSystems[]' | jq "select(.Name==\"${EFS_FILE_SYSTEM_NAME}\")" | jq -r '.FileSystemId')"
               if [ -z "${EFS_FILE_SYSTEM_ID}" ]; then
                 echo "ERROR: variable not set" 1> /etc/efssetup.log
                 exit
               fi
               DIR_SRC="${EC2_AVAIL_ZONE}.${EFS_FILE_SYSTEM_ID}.efs.${EC2_REGION}.amazonaws.com"
               mount -t nfs4 -o nfsvers=4.1,rsize=1048576,wsize=1048576,soft,timeo=600,retrans=2 "${DIR_SRC}:/" "${DIR_TGT}"
               cp -p "/etc/fstab" "/etc/fstab.back-$(date +%F)"
               echo -e "${DIR_SRC}:/ \t\t ${DIR_TGT} \t\t nfs \t\t nfsvers=4.1,rsize=1048576,wsize=1048576,soft,timeo=600,retrans=2 \t\t 0 \t\t 0" | tee -a /etc/fstab
               docker stop ecs-agent
               /etc/init.d/docker restart
               docker start ecs-agent
