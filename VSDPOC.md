# Motivation

Im Big Data Umfeld erfolgt die Datenverarbeitung häufig nach dem "Teile und Herrsche" (bspw. MapReduce Verfahren) Prinzip, dies bedingt eine verteilte Datenhaltung im Rechencluster. 
Das Hadoop Distributed Filesystem (HDFS) ist ein verteiltes Dateisystem, welches dazu entwickelt wurde auf kostengünstiger "Commodity" Hardware betrieben zu werden.
In diesem PoC wird die Anwendung Apache Spark verprobt. Diese wird standardmäßig mit Client Libraries für Hadoop [publiziert][1]. Da dies impliziert, dass die Integration mit HDFS einen ausgeprägten Reifegrad trägt, werden wir HDFS als Dateisystem für Spark nutzen.

# Architektur

HDFS ist nach einer Master/Slave Architektur aufgebaut und in Java geschrieben. 
Der sogenannte Namenode fungiert als Master und verwaltet den Filesystem Namespace, sowieo die Zugriffe der Clients. Daneben operieren mehrere Datanodes, welche den Speicherplatz bereitstellen und die Anfragen der Clients bedienen.
Eine Datei wird in eine Sequenz von Blöcken aufgeteilt. Eine konfigurierbare Anzahl von Repliken pro Block (Default: 3) werden auf den Datanodes verteilt gespeichert.
Hierdurch wird eine hohe Fehlertoleranz gegenüber Datanode Ausfällen generiert. 
Sollte der Namenode ausfallen, ist der gesamte HDFS Cluster nicht erreichbar - bis die Namenode wieder gestartet wird. Um hier Abhilfe zu schaffen, gibt es seit Hadoop 2.0.0 die Möglichkeit die Namenode als Aktiv/Passiv Pärchen zu clustern. Die Synchronisation der beiden Namenodes erfolgt entweder per NFS oder per Journal Quorum.

In diesem PoC wird der Speicherplatz der Datanodes als k8s HostPath Volume bereitgestellt. 
Um die Namenode im Falle eines k8s Clusternode Ausfalls auch auf einem anderen Node starten zu können, sollte hier ein persistentes Volume auf Ceph (oder sonstigem shared FS) (#TODO) für die Metadaten verwendet werden.

# Einschränkungen

Im Kontext dieses PoCs wird auf die Hochverfügbarkeit der Namenode verzichtet. Die Metadaten der Namenode liegen auf einem hochverfügbaren Volume, sodass die Namenode im Fehlerfall auch auf einem anderen k8s Clusternode hochgefahren werden kann. 
Beim Neustarten der Namenode nimmt sie einen sogenannten Safemode Zustand ein, in dem sie Hearbeats, sowie Blockreports von den im Cluster befindlichen Datanodes empfängt. In dieser Phase werden keine Blöcke repliziert.
Wenn eine bestimmter Schwellwert an gemeldeteten Replikas überschritten ist, wird der Safemode verlassen und der HDFS Cluster arbeitet normal weiter.

# Nutzung

Um den HDFS Cluster auf dem k8s Cluster bereitzustellen, wird Helm benötigt. 

Die k8s Clusternodes müssen mit folgenden Labeln ausgestattet werden, da die k8s Nodes für die Pods der Name- und Datanodes per ndoeSelector ausgewählt werden:

"hdfs_cluster":"namenode" für einen Clusternode, auf dem der Namenode laufen soll

"hdfs_cluster":"datanode" für alle anderen Clusternodes 

Das HostPath Volume zeigt für die Datanodes nach /hdfs-data und für die Namenode nach /hdfs-name.

```
  $ helm repo add incubator  \
      https://kubernetes-charts-incubator.storage.googleapis.com/
  $ helm dependency build charts/hdfs-k8s
  $ helm install -n vrsf-hdfs charts/hdfs-k8s
```


[1]: https://spark.apache.org/downloads.html
