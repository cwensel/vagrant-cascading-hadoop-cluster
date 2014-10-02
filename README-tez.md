This cluster uses hadoop 2.4.1 and tez 0.5.0.

To bring up the cluster and install tez do this:

boot the cluster:

    vagrant up

start hadoop (history server is a work in progress):

    vagrant ssh master
	sudo prepare-cluster.sh

	sudo start-dfs.sh --config $HADOOP_CONF_DIR
	sudo start-yarn.sh --config $HADOOP_CONF_DIR

	sudo hadoop fs -mkdir -p /tmp/hadoop-yarn/staging/history/done
	sudo hadoop fs -chmod -R 777 /tmp/
	sudo hadoop fs -mkdir -p /user/
	sudo hadoop fs -chmod -R 777 /user/
	sudo hadoop fs -mkdir -p /apps/
	sudo hadoop fs -chmod -R 777 /apps/

	sudo yarn-daemon.sh --config $HADOOP_CONF_DIR start historyserver

compile and deploy tez:

    tez deploy

`tez` is a script in `/opt/tools/bin` which will checkout tez 0.5.0 from git into `/vagrant/tez`, compile it with maven
(installed in `/opt/tools/apache-maven-3.2.3/bin/mvn`) and copy it onto HDFS in `/apps/tez-0.5.0`. Note that `/vagrant`
is the checkout directory of the cluster, so you can also compile tez on the host machine and simply run `tez upload` on
the cluster, if that is faster for you.

Now you should be read to go.

The cluster is already configured to use tez. You can cofirm that it works like so:

    hadoop jar /opt/hadoop-2.4.1/share/hadoop/mapreduce/hadoop-mapreduce-client-jobclient-2.4.1-tests.jar sleep -mt 1 -rt 1 -m 1 -r 1

Web Interfaces:

* http://master.local:8088/cluster/apps
* http://master.local:50070/dfshealth.jsp
* http://master.local:8188/applicationhistory !currently not working

stop/restart hadoop:

	sudo yarn-daemon.sh --config $HADOOP_CONF_DIR stop historyserver
	sudo stop-yarn.sh --config $HADOOP_CONF_DIR
	sudo stop-dfs.sh --config $HADOOP_CONF_DIR
	for host in master hadoop1 hadoop2 hadoop3; do vagrant ssh $host --command  'sudo rm -rf /srv/hadoop' ; done
	vagrant provision
	
Notes:

This deployment is by default reliant on VMWare, VirtualBox settings are still retained, just comment out the relevant bits.

VM slave memory settings are 2G.

VMWare Fusion wants you to edit the /etc/hosts file and add the .local machines its hosting (sudo dscacheutil -flushcache)

Current vms have no password for root.

To get access to the application logs (from sftp as user vagrant), after a run

	for host in master hadoop1 hadoop2 hadoop3; do vagrant ssh $host --command  'sudo chmod -R 777 /tmp/hadoop-root/ /opt/hadoop-2.4.1/logs/' ; done

Or to have them copied to the local shared folder

	for host in master hadoop1 hadoop2 hadoop3; do vagrant ssh $host --command  "sudo mkdir -p /vagrant/machines/$host/; sudo cp -Ru /tmp/hadoop-root/ /opt/hadoop-2.4.1/logs/ /vagrant/machines/$host/" ; done

Or on master, call (log aggregation is currently enabled by default)
	
	yarn logs -applicationId <application ID>

To grab the stack traces for all running TezChild vms

	for host in master hadoop1 hadoop2 hadoop3; do vagrant ssh $host --command  "sudo mkdir -p /vagrant/stacks/$host; sudo jps | grep TezChild | cut -d\" \" -f1 | xargs -n 1 -I @ sh -c \"sudo jstack @ > /vagrant/stacks/$host/@.txt\" " ; done

