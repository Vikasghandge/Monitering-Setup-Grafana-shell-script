# Monitering-Setup-Grafana-shell-script
This Repo include automated setup of prometheus,grafana,and node-exporter

in above script i have for scripts 
1) is for setup grafana
2) then prometheus and add target into that
3) setup the node exporter

   first of all login into your monitering means that server which you want to moniter.
   add node_exporter script it will setup the node exporter which will collect the matrics and ready to export to the prometheus server.

   then go and setup the server for the prometheus and grafana you can keep single server for the both.
   first setup the prometheus run that script which will setup the prometheus i will prompt for ip address and port number to add in prometheus target
   so add your server ip which you want to moniter or where u installed node-exporter ex- 192.168.0.0:9100


   then run the grafna script which will configure the grafana.
