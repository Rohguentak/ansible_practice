# ansible_practice
vm으로 cluster 구성 후 ansible로 pbspro 설치(local repository)
------------------------------------------------------------------------------------------------------------------------------------
      -os: centos7.7

      -pbspro 버전: pbspro-18.1.4

      -구성 #1: pbs-host 서버 1대, 계산 노드 2대 , nfs 서버 1대

      -구성 #2: pbs-host 서버 2대, 계산 노드 1대 , nfs 서버 1대 (failover 구성)


   centos 서버 구축 및 ssh 접속 설정
   ------------------------------------------------------------------------------------------------------------------------------------
       1. vagrant 사용해서 vm 4대 구성
         - vagrant up
         - vagrant ssh pbs-nfs                       //pbs-host에 접속


      2. ssh를 이용하여 pbs-nfs에서 password없이 pbs components에 접속가능하게 함
         - ssh-keygen
         - ssh-copy-id 172.28.128.11 하거나 vi /etc/hosts에 172.28.128.11 pbs-host를 추가한 뒤 ssh-copy-id pbs-host 실행
   
   
   
   
   
   nfs로 home directory 공유
   ------------------------------------------------------------------------------------------------------------------------------------



   nfs-server 
   ----------
      #yum install -y rpcbind nfs-utils
      #systemctl enable nfs
      #systemctl enable nfslock
      #systemctl enable rpcbind
      
      #systemctl start rpcbind
      #systemctl start nfslock
      #systemctl start nfs 

      #rpcinfo -p localhost                                  //실행되는지 확인
      #mkdir /data
      #vi /etc/exports
      /home/vagrant *(rw,no_root_squash,sync)
      /data *(rw,no_root_squash,sync)
      #sudo service nfs restar


   pbs설치 작업(failover구성시)
   ----------------
            #sudo yum install -y git epel-release
            #sudo yum install -y ansible
            #git clone https://github.com/Rohguentak/ansible_practice
            #cd ansible_practice
            #cp pbs* ~/
            #cp nfs* ~/
            #cd ~/
            #sudo vi /etc/ansible/hosts
            [pbs-host]
            pbshost[1:2]

            [pbs-mom]
            pbsmom
            
            #ansible-playbook nfs_client.yml //mount완료
            nfs-server에서 수행
            #yum install -y gcc make rpm-build libtool hwloc-devel libX11-devel libXt-devel libedit-devel libical-devel ncurses-devel perl postgresql-devel python-devel tcl-devel  tk-devel swig expat-devel openssl-devel libXext libXft wget postgresql-server rpmdevtools
            #rpmdev-setuptree  //vagrant 계정으로 만들것
            #wget https://github.com/PBSPro/pbspro/releases/download/v18.1.4/pbspro-18.1.4.tar.gz
            #tar -xpvf pbspro-18.1.4.tar.gz
            #mv pbspor-18.1.4.tar.gz ~/rpmbuild/SOURCES
            #cd pbspro-18.1.4

            #./autogen.sh                       //configure script와 Makefile들 생성
            #./configure                        //설치 환경 설정 ex)--prefix= 로 pbs_exec의 위치, --with-pbs-server-home= 으로 pbs_home의  위치 설정 가능
            #make
            #cp pbspro.spec ~/rpmbuild/SPECS
            #cd ~/rpmbuild/SPECS
            #rpmbuild -ba pbspro.spec
            
            ---------------------
            로컬 리포지토리 만들기
            ---------------------
            
            #sudo yum install httpd
            #sudo systemctl enable httpd
            #sudo systemctl start httpd //apache server 사용
            #sudo yum install createrepo yum-utils
            #sudo mkdir –p /var/www/html/repos/{base,centosplus,extras,updates}
            #sudo reposync -g -l -d -m --repoid=base --newest-only --download-metadata --download_path=/var/www/html/repos/
            #sudo reposync -g -l -d -m --repoid=centosplus --newest-only --download-metadata --download_path=/var/www/html/repos/
            #sudo reposync -g -l -d -m --repoid=extras --newest-only --download-metadata --download_path=/var/www/html/repos/
            #sudo reposync -g -l -d -m --repoid=updates --newest-only --download-metadata --download_path=/var/www/html/repos/
            #sudo createrepo /var/www/html
            
            ----------------
            client에 root로 접속
            ----------------
            
            #mv /etc/yum.repos.d/*.repo /tmp/
            #vi /etc/yum.repos.d/remote.repo   //text editor로 수정해야함
            [remote]

            name=RHEL Apache

            baseurl=http://192.168.1.10

            enabled=1

            gpgcheck=0
            
            ----------------
            nfs_ser에 접속
            ----------------
            
            #ansible-playbook pbs-install.yml

   
  
   
            
            
   test failover
   -------------
            # pbsnodes -a
            pbsnodes: Server has no node list
            # qstat -B
            Server             Max   Tot   Que   Run   Hld   Wat   Trn   Ext Status
            ---------------- ----- ----- ----- ----- ----- ----- ----- ----- -----------
            pbs-host1            0     0     0     0     0     0     0     0 Active
            # qmgr
            Max open servers: 49
            Qmgr: p s
            #
            # Create queues and set their attributes.
            #
            #
            # Create and define queue workq
            #
            create queue workq
            set queue workq queue_type = Execution
            set queue workq enabled = True
            set queue workq started = True
            #
            # Set server attributes.
            #
            set server scheduling = True
            set server default_queue = workq
            set server log_events = 511
            set server mail_from = adm
            set server query_other_jobs = True
            set server resources_default.ncpus = 1
            set server default_chunk.ncpus = 1
            set server scheduler_iteration = 600
            set server resv_enable = True
            set server node_fail_requeue = 310
            set server max_array_size = 10000
            set server pbs_license_min = 0
            set server pbs_license_max = 2147483647
            set server pbs_license_linger_time = 31536000
            set server eligible_time_enable = False
            set server max_concurrent_provision = 5
            
            #echo "sleep 200" | qsub                               //root계정에서 job제출하면 안됨
            11.pbs-host                                      //job id
            #qstat -a
            pbs-host:
                                                                        Req'd  Req'd   Elap
            Job ID          Username Queue    Jobname    SessID NDS TSK Memory Time  S Time
            --------------- -------- -------- ---------- ------ --- --- ------ ----- - -----
            11.pbs-host     vagrant  workq    STDIN       18685   1   1    --    --  R 00:00

            #ssh pbs-mom
            #ps -ef | grep sleep

            vagrant  18707 18706  0 07:06 ?        00:00:00 sleep 200               //host에서 제출한 job이 
            vagrant  18734 18712  0 07:10 pts/0    00:00:00 grep sleep              //스케줄링 됨
            #sudo /etc/init.d/pbs stop                //pbs-host1 stop
            
            #ps -ef | grep pbs            //pbs-host2에서 실행
            #root      2474     1  0 05:37 ?        00:00:00 /sbin/dhclient -H pbs-host2 -1 -q -cf /etc/dhcp/dhclient-eth0.conf -lf /var/lib/dhclient/dhclient-eth0.leases -pf /var/run/dhclient-eth0.pid eth0
            #root      9320     1  0 06:45 ?        00:00:00 /opt/pbs/sbin/pbs_comm
            #root      9337     1  0 06:45 ?        00:00:00 /opt/pbs/sbin/pbs_server.bin
            
            #ps -ef | grep pbs
            #root      2474     1  0 05:37 ?        00:00:00 /sbin/dhclient -H pbs-host2 -1 -q -cf /etc/dhcp/dhclient-eth0.conf -lf /var/lib/dhclient/dhclient-eth0.leases -pf /var/run/dhclient-eth0.pid eth0
            #root      9320     1  0 06:45 ?        00:00:00 /opt/pbs/sbin/pbs_comm
            #root      9337     1  0 06:45 ?        00:00:00 /opt/pbs/sbin/pbs_server.bin
            #root      9528     1  0 07:01 ?        00:00:00 /opt/pbs/sbin/pbs_ds_monitor monitor
            #postgres  9556     1  0 07:01 ?        00:00:00 /usr/bin/postgres -D /data/pbs/datastore -p 15007          //일정시간이후 pbs-host1의 db가 pbs-host2로 넘어온것을 확인
            #postgres  9724  9556  0 07:01 ?        00:00:00 postgres: postgres pbs_datastore 127.0.0.1(49662) idle
            #root      9730     1  0 07:01 ?        00:00:00 /opt/pbs/sbin/pbs_sched
            
            

            
    
   
