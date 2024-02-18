import PySimpleGUI as sg
import sys, os, string, threading, subprocess
import paramiko, vlc, time
import yaml
import signal,subprocess
from datetime import datetime 
import xlsxwriter

from bokeh.plotting import figure, save, output_file
import numpy as np

threads = []
temp=None
global conf
conf = None
available_ip = None
cur_datetime_obj=datetime.now()
cur_filename = cur_datetime_obj.strftime("%d_%H_%M")
rowindex = 1
timeout_cal=0

ping_destination = 'google.com'
vlc_streaming_url = 'rtp://239.0.0.10:5004'
youtube_url = '"https://www.youtube.com/watch?v=86YLFOog4GM"'
wget_url = 'https://speed.hetzner.de/10GB.bin'

timestamp = []
traffic_monitor_duration = 10

def waiting():
	global timeout_cal
	time.sleep(traffic_overall_duration)
	timeout_cal=1
	if timeout_cal == 1:
		print('timeout call exeeceded\n')
		terminate_process(conf,temp,available_ip, terminate_value = 1)

'''     To create a excel sheet and to write headings    ''' 

workbook = xlsxwriter.Workbook("traffic_output"+cur_filename+".xlsx")

bold_format = workbook.add_format({'bold':True})
cell_format = workbook.add_format()
cell_format.set_text_wrap()
cell_format.set_align('top')
cell_format.set_align('left=')
          
worksheet = workbook.add_worksheet("Client_logs")
worksheet.set_column(0,0,width=20)
worksheet.set_column(1,1,width=30)
worksheet.set_column(2,2,width=30)
worksheet.set_column(3,3,width=20)
worksheet.set_column(4,4,width=30)

worksheet.write("A1", 'Client IP',bold_format)
worksheet.write("B1",'No of packets transmitted',bold_format)
worksheet.write("C1",'No of packets received',bold_format)
worksheet.write("D1",'Packet_loss',bold_format)
worksheet.write("E1",'Iperf_Throughput(Mbits/sec)',bold_format)

kill_iperf = subprocess.getoutput("killall iperf")               #command to kill iperf if that application exists 


''' THREAD FUNCTION TO EXECUTE/TERMINATE PROCESS '''

def ssh_to_available_clients(available_ip, passwd, usernm):

	try:
		if (temp == '1'):                       #ssh if user selects nmap option
			ssh = paramiko.SSHClient()
			ssh.set_missing_host_key_policy(paramiko.AutoAddPolicy())
			ssh.connect(available_ip, username='lekha.t', password='vvdn@123')
		elif (temp == '2'):                     #ssh if user selects config file option
			ssh = paramiko.SSHClient()
			ssh.set_missing_host_key_policy(paramiko.AutoAddPolicy())
			ssh.connect(available_ip, username=usernm, password=passwd)
          
	except Exception as e:
		return_value = 'SSHConnection Error %s' % e 

def find_client_os_type(ssh):

	''' script to find OS type of device '''    
	find_os = 'uname -a'       
	find_os_type_linux = ['None']                                   #command to find os type of client device
	stdin, stdout, stderr = ssh.exec_command(find_os)
	find_os_type_linux = stdout.readlines()
	if len(find_os_type_linux) == 0:
		find_os_type_linux.append('dummy')
          
	''' CONDITION FOR LINUX CLIENT '''
	if 'Linux' in find_os_type_linux[0]:
		os_model = 'Linux'
		
	elif 'MacBook' in find_os_type_linux[0]:
		os_model = 'MacBook'
          
	else:
		os_model = 'Windows'
		
	return os_model

def hostname_querysession_windows(ssh):
	stdin1,host_name,stderr1 = ssh.exec_command("hostname")                
	hostnm = host_name.readlines()[0].split('\r')[0]
	
	#stdin2, query_ses, stderr2 = ssh.exec_command("query session | findstr console")
	#query_session = query_ses.readlines()
	#a = query_session[0].split(' ')
	a = ['1', '22']
	for i in a:
		if len(i)==1:
			query_session = i
			print('querysession', query_session)
			
	return hostnm, query_session


def get_adb_device_ids():
	ids = subprocess.check_output("adb devices", shell = True)
	decode_ids = ids.decode("utf-8")
	individual_ids = decode_ids.split('\n')
	adb_list = []
	for index, i in enumerate(individual_ids):
		if index != 0:
			adb_ids = i.split('\t')
			if len(adb_ids) == 2:
				adb_list.append(adb_ids[0])		
	return(adb_list)

def ping_plot_live_update (input_dict, timestamp):
	
	os.system('rm ping_plot_Traffic_report.html')
	print(input_dict, timestamp)
	for index, ips in enumerate(input_dict):

		clinet_ip = ips
		client_count = 'p' + str(index)
		y = input_dict[ips]
		x = timestamp

		client_count = figure(width=800, height=400, x_axis_label='Traffic time (Minutes)', y_axis_label = 'Ping loss(%)')

		client_count.line(x, y, legend_label=ips, color = 'red')

		output_file('ping_plot_Traffic_report.html', title = 'Traffic Generator')
		
		save(client_count)

def iperf_plot_live_update (input_dict, timestamp):
	
	os.system('rm iperf_plot_Traffic_report.html')
	print(input_dict, timestamp)
	for index, ips in enumerate(input_dict):

		clinet_ip = ips
		client_count = 'p' + str(index)
		y = input_dict[ips]
		x = timestamp

		client_count = figure(width=800, height=400, x_axis_label='Traffic time (Minutes)', y_axis_label = 'Throughput(%)')

		client_count.line(x, y, legend_label=ips, color = 'red')

		output_file('iperf_plot_Traffic_report.html', title = 'Traffic Generator')
		
		save(client_count)

def linux_traffic_execute(ssh, port_no, available_ip):

	youtube_command = "export DISPLAY=:0; google-chrome " + youtube_url
	vlc_command = 'export DISPLAY=:0; vlc ' + vlc_streaming_url
	ping_command = "export DISPLAY=:0; gnome-terminal -- /bin/sh -c 'ping -w "+str(traffic_monitor_duration)+ " " + ping_destination + " >> log_ping_"+cur_filename+".txt & tail -f log_ping_"+cur_filename+".txt; exec bash'"
	wget_command = "export DISPLAY=:0; gnome-terminal -- /bin/sh -c 'wget  " + wget_url + "; exec bash'"
	iperf_server = "export DISPLAY=:0; gnome-terminal -- /bin/sh -c 'iperf -s -p "+str(port_no)+" -i 1 ; exec bash'"
	iperf_command = "export DISPLAY=:0; gnome-terminal -- /bin/sh -c 'iperf -c " +str(server_ip) + " -i1 -t"+str(traffic_monitor_duration)+" -p "+str(port_no)+" >> log_iperf_"+str(cur_filename)+".txt & tail -f log_iperf_"+str(cur_filename)+".txt ; exec bash'"
	
	print(iperf_command)
	
	if 'youtube' in traffic_to_trigger:
		stdin3, stdout3, stderr3 = ssh.exec_command(youtube_command)
		print('Executing youtube video in ', available_ip)  
                   
	if 'vlc' in traffic_to_trigger:
		stdin1, stdout1, stderr1 = ssh.exec_command(vlc_command)
		print('Executing vlc in ', available_ip)

	if 'wget' in traffic_to_trigger:
		stdin1, stdout1, stderr1 = ssh.exec_command(wget_command)
		print('Executing wget in ', available_ip)

	if 'ping' in traffic_to_trigger:
		stdin2, stdout2, stderr2 = ssh.exec_command(ping_command)
		print('Executing ping in ', available_ip)

	if 'iperf' in traffic_to_trigger:
		os.system(iperf_server)
		print("iperf server for "+str(available_ip)+" is running on server side.....")
		
		stdin4, stdout4, stderr4 = ssh.exec_command(iperf_command)
		print('Executing iperf client in ', available_ip)

def linux_traffic_resume(ssh, port_no, available_ip):

	youtube_command = "export DISPLAY=:0; google-chrome " + youtube_url
	vlc_command = 'export DISPLAY=:0; vlc ' + vlc_streaming_url
	ping_command = "export DISPLAY=:0; gnome-terminal -- /bin/sh -c 'ping -w "+str(traffic_monitor_duration)+ " " + ping_destination + " >> log_ping_"+cur_filename+".txt & tail -f log_ping_"+cur_filename+".txt; exec bash'"
	wget_command = "export DISPLAY=:0; gnome-terminal -- /bin/sh -c 'wget  " + wget_url + "; exec bash'"
	iperf_server = "export DISPLAY=:0; gnome-terminal -- /bin/sh -c 'iperf -s -p "+str(port_no)+" -i 1 ; exec bash'"
	iperf_command = "export DISPLAY=:0; gnome-terminal -- /bin/sh -c 'iperf -c " +str(server_ip) + " -i1 -t"+str(traffic_monitor_duration)+" -p "+str(port_no)+" >> log_iperf_"+str(cur_filename)+".txt & tail -f log_iperf_"+str(cur_filename)+".txt ; exec bash'"
	
	if 'youtube' in traffic_to_trigger:
		stdin3, stdout4, stderr3 = ssh.exec_command('killall chrome')
		stdin3, stdout3, stderr3 = ssh.exec_command(youtube_command)
		print('Executing youtube video in ', available_ip)  
                   
	if 'vlc' in traffic_to_trigger:
		stdin1, stdout1, stderr1 = ssh.exec_command(vlc_command)
		print('Executing vlc in ', available_ip)

	if 'wget' in traffic_to_trigger:
		stdin1, stdout1, stderr1 = ssh.exec_command(wget_command)
		print('Executing wget in ', available_ip)
		
	if 'ping' in traffic_to_trigger:
		stdin2, stdout2, stderr2 = ssh.exec_command(ping_command)
		print('Executing ping in ', available_ip)

	if 'iperf' in traffic_to_trigger:
		#os.system(iperf_server)
		print("iperf server for "+str(available_ip)+" is running on server side.....")
		
		stdin4, stdout4, stderr4 = ssh.exec_command(iperf_command)
		print('Executing iperf client in ', available_ip)
		
def linux_traffic_terminate(ssh, available_ip):

	worksheet.write('A' + str(rowindex),available_ip,cell_format)
	print('@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@')
	termination_command_vlc = 'killall vlc'
	termination_command_youtube = 'killall chrome'
	termination_command_cmdprompt = 'killall gnome-terminal-server'                        # command to close the terminal
	termination_command_ping = 'killall ping'                        # command to close the terminal
	termination_command_iperf = 'killall iperf'                        # command to close the terminal
	
	
	if 'vlc' in traffic_to_trigger:
		stdin1, stdout1, stderr1 = ssh.exec_command(termination_command_vlc)
		
	if 'youtube' in traffic_to_trigger:
		print('@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@terminate')
		stdin2, stdout2, stderr2 = ssh.exec_command(termination_command_youtube)
		
	if 'iperf' in traffic_to_trigger:
		stdin3, stdout3, stderr3 = ssh.exec_command(termination_command_iperf)
		stdin3, stdout3, stderr3 = ssh.exec_command(termination_command_cmdprompt)
		

		throughput_iperf = "cat log_iperf_"+cur_filename+".txt | awk '{print $7}' | tail -n 1"
		print(throughput_iperf)
		stdin5, stdout5, stder5 = ssh.exec_command(throughput_iperf)
		throughput = stdout5.readlines()[0]
		print('Throughput of '+available_ip+' is : ',throughput)
		
		worksheet.write('E' + str(rowindex),throughput,cell_format)
		
		iperf_traffic_output_dict[available_ip].append(throughput)
		
	if 'ping' in traffic_to_trigger:
		stdin3, stdout3, stderr3 = ssh.exec_command(termination_command_ping)
		time.sleep(3)
		stdin3, stdout3, stderr3 = ssh.exec_command(termination_command_cmdprompt)
		
		no_transmitted = "cat log_ping_"+cur_filename+".txt | grep -i transmitted | awk '{print $1}'"
		stdin3,stdout3,stderr3 = ssh.exec_command(no_transmitted)
		no_pkt_transmitted = stdout3.readlines()[0]

		no_received = "cat log_ping_"+cur_filename+".txt | grep -i transmitted | awk '{print $4}'"
		stdin5,stdout5,stderr5 = ssh.exec_command(no_received)
		no_pkt_received = stdout5.readlines()[0]
		
		termination_command_ping_pack = "cat log_ping_"+cur_filename+".txt | grep -i 'packet loss' | awk '{print $6}'"
		stdin3, stdout3, stder3 = ssh.exec_command(termination_command_ping_pack)
		packet_loss = stdout3.readlines()[0].split('\n')[0]
	
		print("The packet loss for "+available_ip+" is :",packet_loss)
		
		worksheet.write('D' + str(rowindex),packet_loss,cell_format)
		worksheet.write('B' + str(rowindex),no_pkt_transmitted,cell_format)
		worksheet.write('C' + str(rowindex),no_pkt_received,cell_format)
		
		packet_loss_value = packet_loss.split('%')
		ping_traffic_output_dict[available_ip].append(int(packet_loss_value[0]))



def windows_traffic_execute(ssh, passwd, usernm, port_no, hostnm, query_session, available_ip):

	vlc_command = "psexec -u "+usernm+" -p "+passwd+ ' -s -i ' +query_session+ " vlc " + vlc_streaming_url
	utube_command = "psexec -u "+usernm+" -p "+passwd+ ' -s -i ' +query_session+" chrome " + youtube_url	
	ping_command = 'psexec '+r'\\'+hostnm+' -u '+usernm+' -p '+passwd+' -s -i '+query_session+' cmd /c ping -n '+str(traffic_monitor_duration)+' ' + ping_destination
	wget_command = 'psexec '+r'\\'+hostnm+' -u '+usernm+' -p '+passwd+' -s -i '+query_session+' cmd /c wget ' + wget_url
	iperf_command = "export DISPLAY=:0; gnome-terminal -- /bin/sh -c 'iperf -s -p "+str(port_no)+" -i 1 ; exec bash'"    #to run iperf server
	iperf_client = 'psexec '+r'\\'+hostnm+' -u '+usernm+' -p '+passwd+' -i '+query_session+' cmd /c iperf -c '+str(server_ip)+' -i 1 -t '+str(traffic_monitor_duration)+' -p '+str(port_no) #to run iperf client in client device
	
	print(utube_command, iperf_client)

	if 'vlc' in traffic_to_trigger:
		stdin3, stdout3, stderr3 = ssh.exec_command(vlc_command)
		print('vlc command is executing in '+available_ip)

	if 'youtube' in traffic_to_trigger:
		print(utube_command)
		stdin5, stdout5, stderr5 = ssh.exec_command(utube_command)
		print('utube command is executing in '+available_ip)
            
	if 'wget' in traffic_to_trigger:
		stdin1, stdout1, stderr1 = ssh.exec_command(wget_command)
		print('Executing wget in ', available_ip)
		
	if 'ping' in traffic_to_trigger:
		stdin5, stdout5, stderr5 = ssh.exec_command(ping_command)
		print('Ping command is executing in '+available_ip, ping_command)

	if 'iperf' in traffic_to_trigger:
		os.system(iperf_command)
		print('iperf for '+str(available_ip)+' is running in server side....')
            
		stdin4,stdout4,stderr4 = ssh.exec_command(iperf_client)
		print('iperf for '+str(available_ip)+' is running in client side....', iperf_client)
		
def windows_traffic_resume(ssh, passwd, usernm, port_no, hostnm, query_session, available_ip):

	vlc_command = "psexec -u "+usernm+" -p "+passwd+ ' -s -i ' +query_session+ " vlc " + vlc_streaming_url
	utube_command = "psexec -u "+usernm+" -p "+passwd+ ' -s -i ' +query_session+" chrome " + youtube_url	
	ping_command = 'psexec '+r'\\'+hostnm+' -u '+usernm+' -p '+passwd+' -s -i '+query_session+' cmd /c ping -n '+str(traffic_monitor_duration)+' ' + ping_destination
	wget_command = 'psexec '+r'\\'+hostnm+' -u '+usernm+' -p '+passwd+' -s -i '+query_session+' cmd /c wget ' + wget_url
	iperf_command = "export DISPLAY=:0; gnome-terminal -- /bin/sh -c 'iperf -s -p "+str(port_no)+" -i 1 ; exec bash'"    #to run iperf server
	iperf_client = 'psexec '+r'\\'+hostnm+' -u '+usernm+' -p '+passwd+' -s -i '+query_session+' cmd /c iperf -c '+str(server_ip)+' -i 1 -t '+str(traffic_monitor_duration)+' -p '+str(port_no) #to run iperf client in client device

	if 'vlc' in traffic_to_trigger:
		stdin3, stdout3, stderr3 = ssh.exec_command(vlc_command)
		print('vlc command is executing in '+available_ip)

	if 'youtube' in traffic_to_trigger:
		print(utube_command)
		stdin5, stdout5, stderr5 = ssh.exec_command(utube_command)
		print('utube command is executing in '+available_ip)

	if 'wget' in traffic_to_trigger:
		stdin1, stdout1, stderr1 = ssh.exec_command(wget_command)
		print('Executing wget in ', available_ip)
		
	if 'ping' in traffic_to_trigger:
		stdin5, stdout5, stderr5 = ssh.exec_command(ping_command)
		print('Ping command is executing in '+available_ip, ping_command)

	if 'iperf' in traffic_to_trigger:
		os.system(iperf_command)
		print('iperf for '+str(available_ip)+' is running in server side....')
            
		stdin4,stdout4,stderr4 = ssh.exec_command(iperf_client)
		print('iperf for '+str(available_ip)+' is running in client side....', iperf_client)
            
def windows_traffic_terminate(ssh, available_ip):

	worksheet.write('A' + str(rowindex),available_ip,cell_format)

	termination_command_vlc = 'Taskkill /IM vlc.exe /F'
	termination_command_commandprompt = 'Taskkill /F /IM "cmd.exe"'
	termination_command_chrome = 'Taskkill /F /IM chrome.exe'
		
	ping_loss = "cat C:\Windows\System32\ping_log_"+cur_filename+".txt | grep loss | awk '{print $11}'"      
	ping_no_transmitted = "cat C:\Windows\System32\ping_log_"+cur_filename+".txt | grep -i sent | awk '{print $4}'"
	ping_no_received = "cat C:\Windows\System32\ping_log_"+cur_filename+".txt | grep -i sent | awk '{print $7}'"
	iperf_tput = "cat C:\Windows\System32\iperf_log_"+cur_filename+".txt | grep receiver | awk '{print $7,$8}'"
	
	if 'vlc' in traffic_to_trigger:
		stdin1, stdout1, stderr1 = ssh.exec_command(termination_command_vlc)
		
	if 'youtube' in traffic_to_trigger:
		stdin3, stdout3, stderr3 = ssh.exec_command(termination_command_chrome)
		

	if 'iperf' in traffic_to_trigger:
		stdin2,stdout2,stderr2 = ssh.exec_command(termination_command_commandprompt)
			
		stdin2,throughpu,stderr2 = ssh.exec_command(iperf_tput)
		throughput = throughpu.readlines()[0]
		print("The Throughput of "+available_ip+" is : ",throughput)                
		
		worksheet.write('E' + str(rowindex),throughput,cell_format)
		
		iperf_traffic_output_dict[available_ip].append(throughput)
	
	if 'ping' in traffic_to_trigger:
		stdin2,stdout2,stderr2 = ssh.exec_command(termination_command_commandprompt)
		
		stdin1, ping, stderr1 = ssh.exec_command(ping_loss)
		ping_out = ping.readlines()[0].split('(')[1]
		print("The packet loss of "+available_ip+" is ",ping_out)           

		stdin3,ping_pkt_tx,stderr3 = ssh.exec_command(no_transmitted)
		no_pkt_transmitted = ping_pkt_tx.readlines()[0].split(',')[0]
	
		stdin4,ping_pkt_rx,stderr4 = ssh.exec_command(no_received)
		no_pkt_received = ping_pkt_rx.readlines()[0].split(',')[0]
		
		worksheet.write('D' + str(rowindex),ping_out,cell_format)
		worksheet.write('B' + str(rowindex),no_pkt_transmitted,cell_format)
		worksheet.write('C' + str(rowindex),no_pkt_received,cell_format)
			
		packet_loss_value = packet_loss.split('%')
		ping_traffic_output_dict[available_ip].append(int(packet_loss_value[0]))


def macbook_traffic_execute(ssh, port_no, available_ip):

	youtube_command = 'open -a "Google Chrome" ' + youtube_url 
	vlc_command = 'open -a "VLC" ' + vlc_streaming_url
	ping_command = "ping -c "+str(traffic_monitor_duration)+" " + ping_destination + " > log_ping_"+cur_filename+".txt"
	ping_command_display = "echo /usr/bin/tail -f log_ping_" + cur_filename + ".txt > /tmp/tmp.sh;chmod a+x /tmp/tmp.sh;open -n -a Terminal /tmp/tmp.sh"
	iperf_server = "export DISPLAY=:0; gnome-terminal -- /bin/sh -c 'iperf -s -p "+str(port_no)+" -i 1 ; exec bash'"
	iperf_command = "/usr/local/bin/iperf -c " +str(server_ip) + " -i1 -t"+str(traffic_monitor_duration)+" -p "+str(port_no)+" > log_iperf_"+str(cur_filename)+".txt &"
	iperf_command_display = "echo /usr/bin/tail -f log_iperf_"+ cur_filename+ ".txt > /tmp/tmp.sh;chmod a+x /tmp/tmp.sh;open -n -a Terminal /tmp/tmp.sh"
	
	if 'youtube' in traffic_to_trigger:
		stdin3, stdout3, stderr3 = ssh.exec_command(youtube_command)
		print('Executing youtube video in ', available_ip, youtube_command)  
                   
	if 'vlc' in traffic_to_trigger:
		stdin1, stdout1, stderr1 = ssh.exec_command(vlc_command)
		print('Executing vlc in ', available_ip)

	if 'ping' in traffic_to_trigger:
		stdin2, stdout2, stderr2 = ssh.exec_command(ping_command)
		stdin3, stdout3, stderr3 = ssh.exec_command(ping_command_display)
		print('Executing ping in ', available_ip)

	if 'iperf' in traffic_to_trigger:
		os.system(iperf_server)
		print("iperf server for "+str(available_ip)+" is running on server side.....")	
		stdin4, stdout4, stderr4 = ssh.exec_command(iperf_command)
		stdin5, stdout5, stderr5 = ssh.exec_command(iperf_command_display)
		print('Executing iperf client in ', available_ip)
		
def macbook_traffic_resume(ssh, port_no, available_ip):

	youtube_command = 'open -a "Google Chrome" ' + youtube_url
	vlc_command = 'open -a "VLC" '+ vlc_streaming_url
	ping_command = "ping -c "+str(traffic_monitor_duration)+" " + ping_destination + " > log_ping_"+cur_filename+".txt"
	ping_command_display = "echo /usr/bin/tail -f log_ping_" + cur_filename + ".txt > /tmp/tmp.sh;chmod a+x /tmp/tmp.sh;open -n -a Terminal /tmp/tmp.sh"
	iperf_server = "export DISPLAY=:0; gnome-terminal -- /bin/sh -c 'iperf -s -p "+str(port_no)+" -i 1 ; exec bash'"
	iperf_command = "/usr/local/bin/iperf -c " +str(server_ip) + " -i1 -t"+str(traffic_monitor_duration)+" -p "+str(port_no)+" > log_iperf_"+str(cur_filename)+".txt &"
	iperf_command_display = "echo /usr/bin/tail -f log_iperf_"+ cur_filename+ ".txt > /tmp/tmp.sh;chmod a+x /tmp/tmp.sh;open -n -a Terminal /tmp/tmp.sh"
	
	if 'youtube' in traffic_to_trigger:
		stdin3, stdout3, stderr3 = ssh.exec_command(youtube_command)
		print('Executing youtube video in ', available_ip)  
                   
	if 'vlc' in traffic_to_trigger:
		stdin1, stdout1, stderr1 = ssh.exec_command(vlc_command)
		print('Executing vlc in ', available_ip)

	if 'ping' in traffic_to_trigger:
		stdin2, stdout2, stderr2 = ssh.exec_command(ping_command)
		stdin3, stdout3, stderr3 = ssh.exec_command(ping_command_display)
		print('Executing ping in ', available_ip)

	if 'iperf' in traffic_to_trigger:
		os.system(iperf_server)
		print("iperf server for "+str(available_ip)+" is running on server side.....")	
		stdin4, stdout4, stderr4 = ssh.exec_command(iperf_command)
		stdin5, stdout5, stderr5 = ssh.exec_command(iperf_command_display)
		print('Executing iperf client in ', available_ip)

def macbook_traffic_terminate(ssh, available_ip):

	worksheet.write('A' + str(rowindex),available_ip,cell_format)
	
	termination_command_vlc = 'killall "VLC"'
	termination_command_youtube = 'killall "Google Chrome"'
	termination_command_iperf = 'killall "iperf"'                        # command to close the terminal
	termination_command_cmdprompt = 'killall "Terminal"'    
	termination_command_tail = 'killall "tail"'                        # command to close the terminal                    # command to close the terminal
	
	if 'iperf' in traffic_to_trigger:
		stdin, stdout, stderr = ssh.exec_command(termination_command_tail)
		stdin, stdout, stderr = ssh.exec_command(termination_command_iperf)
		stdin3, stdout3, stderr3 = ssh.exec_command(termination_command_cmdprompt)
		

		throughput_iperf = "cat log_iperf_"+cur_filename+"txt | awk '{print $7}' | tail -n 1"
		stdin5, stdout5, stder5 = ssh.exec_command(throughput_iperf)
		throughput = stdout5.readlines()[0]

		print('Throughput of '+available_ip+' is : ',throughput)
		
		worksheet.write('E' + str(rowindex),throughput,cell_format)
			
		iperf_traffic_output_dict[available_ip].append(throughput)
		
		
	if 'ping' in traffic_to_trigger:
		stdin, stdout, stderr = ssh.exec_command(termination_command_tail)
		stdin3, stdout3, stderr3 = ssh.exec_command(termination_command_cmdprompt)
		
		no_transmitted = "cat log_ping_"+cur_filename+".txt | grep -i transmitted | awk '{print $1}'"
		stdin3,stdout3,stderr3 = ssh.exec_command(no_transmitted)
		no_pkt_transmitted = stdout3.readlines()[0]

		no_received = "cat log_ping_"+cur_filename+".txt | grep -i transmitted | awk '{print $4}'"
		stdin5,stdout5,stderr5 = ssh.exec_command(no_received)
		no_pkt_received = stdout5.readlines()[0]
		
		termination_command_ping_pack = "cat log_ping_"+cur_filename+".txt | grep -i 'packet loss' | awk '{print $7}'"
		stdin3, stdout3, stder3 = ssh.exec_command(termination_command_ping_pack)
		packet_loss = stdout3.readlines()[0].split('\n')[0]
	
		print("The packet loss for "+available_ip+" is :",packet_loss)
		
		worksheet.write('D' + str(rowindex),packet_loss,cell_format)
		worksheet.write('B' + str(rowindex),no_pkt_transmitted,cell_format)
		worksheet.write('C' + str(rowindex),no_pkt_received,cell_format)
			
		packet_loss_value = packet_loss.split('%')
		ping_traffic_output_dict[available_ip].append(int(packet_loss_value[0]))
	
	if 'vlc' in traffic_to_trigger:
		stdin1, stdout1, stderr1 = ssh.exec_command(termination_command_vlc)
		
	if 'youtube' in traffic_to_trigger:
		stdin2, stdout2, stderr2 = ssh.exec_command(termination_command_youtube)

	

def android_traffic_execute(port_no, available_id):

	youtube_command = 'adb -s ' + available_id + ' shell am start -a android.intent.action.VIEW -d "vnd.'+ youtube_url + '"'
	vlc_command = 'adb -s ' + available_id + ' shell am start -n org.videolan.vlc/org.videolan.vlc.gui.video.VideoPlayerActivity -e PlaybackMode 1 -d ' + vlc_streaming_url
	print(youtube_command)
	if 'youtube' in traffic_to_trigger:
		os.system(youtube_command)
		print('Executing youtube video in ', available_id)  
                   
	if 'vlc' in traffic_to_trigger:
		os.system(vlc_command)
		print('Executing vlc in ', available_id)
		
def android_traffic_resume(port_no, available_id):

	youtube_command = 'adb -s ' + available_id + ' shell am start -a android.intent.action.VIEW -d "vnd.'+ youtube_url + '"'
	vlc_command = 'adb -s ' + available_id + ' shell am start -n org.videolan.vlc/org.videolan.vlc.gui.video.VideoPlayerActivity -e PlaybackMode 1 -d ' + vlc_streaming_url

	if 'youtube' in traffic_to_trigger:
		os.system(youtube_command)
		print('Executing youtube video in ', available_id)  
                   
	if 'vlc' in traffic_to_trigger:
		os.system(vlc_command)
		print('Executing vlc in ', available_id)


def android_traffic_terminate(available_id):
	worksheet.write('A' + str(rowindex),available_id,cell_format)

	termination_command_vlc = 'adb -s ' + available_id + ' shell am force-stop org.videolan.vlc'
	termination_command_youtube = 'adb -s ' + available_id + ' shell am force-stop com.google.android.youtube'


	if 'vlc' in traffic_to_trigger:
		os.system(termination_command_vlc)
		
	if 'youtube' in traffic_to_trigger:
		os.system(termination_command_youtube)


def workon(available_ip, traffic_exe_ter,port_no,temp,passwd,usernm,doc,rowindex,traffic_monitor_duration):

	os_model = 'android'

	if (temp == '1'):           
		ssh = paramiko.SSHClient()
		ssh.set_missing_host_key_policy(paramiko.AutoAddPolicy())
		ssh.connect(available_ip, username='netgear', password='Netgear@123')
	elif (temp == '2') and usernm != 'True':                     #ssh if user selects config file option
		ssh = paramiko.SSHClient()
		ssh.set_missing_host_key_policy(paramiko.AutoAddPolicy())
		ssh.connect(available_ip, username=usernm, password=passwd)
		os_model = find_client_os_type(ssh)
	if os_model == 'Linux':
		#print(available_ip , 'is Linux client')
		
		if traffic_exe_ter == 'execute':
			linux_traffic_execute(ssh, port_no, available_ip)
			
		elif traffic_exe_ter == 'terminate':
			linux_traffic_terminate(ssh, available_ip)   
			
		elif traffic_exe_ter == 'resume':
			print(port_no, available_ip)
			linux_traffic_resume(ssh, port_no, available_ip)           

	if os_model == 'Windows':
		print('Client IP Windows: ', available_ip)

		hostnm, query_session = hostname_querysession_windows(ssh)
		
		if traffic_exe_ter == 'execute':
			windows_traffic_execute(ssh, passwd, usernm, port_no, hostnm, query_session, available_ip)		
		
		elif traffic_exe_ter == 'terminate':
			windows_traffic_terminate(ssh, available_ip)  

		elif traffic_exe_ter == 'resume':
			windows_traffic_resume(ssh, port_no, available_ip)  
			
			
	if os_model == 'MacBook':	
		print(available_ip , 'is MacBook client')
		
		if traffic_exe_ter == 'execute':
			macbook_traffic_execute(ssh, port_no, available_ip)
			
		elif traffic_exe_ter == 'terminate':
			macbook_traffic_terminate(ssh, available_ip) 

		elif traffic_exe_ter == 'resume':
			macbook_traffic_resume(ssh, port_no, available_ip) 
			
	if os_model != 'android':	         
		ssh.close() 
	
	if usernm == 'True':	
		print(available_ip , 'is Android client')
		
		if traffic_exe_ter == 'execute':
			android_traffic_execute(port_no, available_ip)
			
		elif traffic_exe_ter == 'terminate':
			android_traffic_terminate(available_ip) 
			
		elif traffic_exe_ter == 'resume':
			android_traffic_resume(available_ip)
	      
	#stdin.flush()  
      
	
''' FUNCTION FOR NMAP SCANNING OPTION '''

def nmap_scanning_input():
	port=2000
	global temp
   
	kill_pre_iperf = subprocess.getoutput("killall iperf")
    
	for ip in available_ip:
     
		#if ip != '6685--QA--POY':
		port+=1
		temp = '1'
		password=None
		usernm = None
		doc = None
		rowindex = None
		t = threading.Thread(target=workon, args=(ip, 'execute', str(port),temp,password,usernm,doc,rowindex,traffic_monitor_duration))           
		t.start()           
		threads.append(t) 
		auto_t = threading.Thread(target=waiting)
		auto_t.start()  

''' FUNCTION FOR CONFIG FILE SCANNING OPTION '''

def config_file_input(doc):
	port = 2000
	global temp
	temp = '2'
	client_list=doc.keys()
	print(client_list)
	kill_pre_iperf = subprocess.getoutput("killall iperf")
	overall_clients = []
	for client in client_list:
		if 'Client' in client:
			client_ip = doc[client]['ip_address']
			overall_clients.append(client_ip)
			print(overall_clients)
	global ping_traffic_output_dict
	ping_traffic_output_dict = {new_list: [] for new_list in overall_clients}
	
	global iperf_traffic_output_dict
	iperf_traffic_output_dict = {new_list: [] for new_list in overall_clients}
	
	
	for client in client_list:
		rowindex=None	
		port = 2000
		if 'Client' not in client:
			usernm = doc['Android_Devices']
			total_android_devices = get_adb_device_ids()
			for available_id in total_android_devices:
				t = threading.Thread(target=workon, args=(available_id, 'execute', port,temp,'none',usernm,doc,rowindex,traffic_monitor_duration))
				t.start()
				threads.append(t) 
				#auto_t = threading.Thread(target=waiting)
				#auto_t.start() 
		
		else:

			available_ip = doc[client]['ip_address']
			passwrd = doc[client]['password']
			usernm = doc[client]['username']
			port_number = client.split('Client')
			port = port + int(port_number[-1])
			t = threading.Thread(target=workon, args=(available_ip, 'execute', port,temp,passwrd,usernm,doc,rowindex,traffic_monitor_duration))
			t.start()
			threads.append(t) 
	auto_t = threading.Thread(target=waiting)
	auto_t.start() 
'''		
	count = 1
	while 1:
		try:
			count = count + 1
			if (timeout_cal):
				raise Exception

			time.sleep(traffic_monitor_duration)
			time.sleep(2)
			#cur_timestamp_obj=datetime.now()
			#cur_timestamp = cur_timestamp_obj.strftime("%d_%H_%M")
			#timestamp.append(cur_timestamp)
			timestamp.append(count)
			for client in client_list:
				rowindex=None	
				port = 2000
				if 'Client' not in client:
					usernm = doc['Android_Devices']
					total_android_devices = get_adb_device_ids()
					for available_id in total_android_devices:
						t = threading.Thread(target=workon, args=(available_id, 'resume', port,temp,'none',usernm,doc,rowindex,traffic_monitor_duration))
						t.start()
						threads.append(t) 
		
				else:
					available_ip = doc[client]['ip_address']
					passwrd = doc[client]['password']
					usernm = doc[client]['username']
					
					port_number = client.split('Client')
					port = port + int(port_number[-1])
					terminate_process(conf,temp,available_ip, terminate_value = 0)

					t = threading.Thread(target=workon, args=(available_ip, 'resume', port,temp,passwrd,usernm,doc,rowindex,traffic_monitor_duration))
					t.start()
					threads.append(t) 
			print(ping_traffic_output_dict)
			print(iperf_traffic_output_dict)
			ping_plot_live_update(ping_traffic_output_dict, timestamp)	
			iperf_plot_live_update(iperf_traffic_output_dict, timestamp)	
		except (Exception,KeyboardInterrupt):
			terminate_process(conf,temp,available_ip, terminate_value = 0)
			break

'''
'''FUNCTION TO TERMINATE PROCESS FOR KEYBOARD INTERRUPT/AUTOMATIC TERMINATE'''

def terminate_process(doc,temp,available_ip, terminate_value = 1):
		global rowindex
		global timeout_cal
		print('Inside Terminate process')
		'''if terminate_value == 1:
			sg.theme('DarkAmber')
			layout = [  [sg.Text('Terminate the Process ')], [sg.Button('YES'), sg.Button('NO')] ]
			window = sg.Window('Window Title', layout)
			event, network_range = window.read()
			if event == 'NO': # if user closes window or clicks cancel
				window.close()
			else:
				window.close()
				print('Terminating the progress')
			sg.mainloop()'''
            		
		if temp == '1':
			for ip in available_ip:
				#if ip != '6685--QA--POY':
				password = None
				usernm = None
				port = None
				doc = None
				rowindex+=1
				traffic_monitor_duration = None
				terminating_t = threading.Thread(target=workon,args=(ip,'terminate',port,temp,password,usernm,doc,rowindex,traffic_monitor_duration))
				terminating_t.start()
				threads.append(terminating_t)
				terminating_t.join()
			workbook.close()
		elif temp == '2':
			client_list=doc.keys()
			for client in client_list:
				port = None
				rowindex+=1
				traffic_monitor_duration = None
				if 'Client' not in client:
					usernm = doc['Android_Devices']
					total_android_devices = get_adb_device_ids()
					for available_id in total_android_devices:
						time.sleep(5)
						t = threading.Thread(target=workon, args=(available_id, 'terminate', port,temp,'none',usernm,doc,rowindex,traffic_monitor_duration))
						t.start()
						threads.append(t) 
				else:	
					available_ip = doc[client]['ip_address']
					passwrd = doc[client]['password']
					usernm = doc[client]['username']
					print('@@###################', available_ip)
					terminating_t = threading.Thread(target=workon, args=(available_ip, 'terminate',port,temp,passwrd,usernm,doc,rowindex,traffic_monitor_duration))
					terminating_t.start()
					threads.append(terminating_t)  
					terminating_t.join()
						
					#workbook.close()
				#break

''' Main code to get input from client and to call workon() and terminate() function '''

if(__name__=="__main__"):  
           
	'''CODE FOR GUI'''
	sg.theme('DarkAmber')


	layout4 = [[sg.Frame(layout=[[
	sg.Checkbox('1. VLC Streaming', default=False,key='vlc'), 
	sg.Checkbox('2. Ping Traffic', default=False,key='ping'),
	sg.Checkbox('3. Iperf Traffic (Uplink)', default=False,key='iperf'),
	sg.Checkbox('4. Youtube Live video streaming', default=False,key='youtube'), 
	sg.Checkbox('5. WGET Download', default=False,key='wget')]], 
	title='Select Traffic to trigger from the Checkbox', relief=sg.RELIEF_SUNKEN, tooltip='Use these to set flags')], 
	[sg.Submit(), sg.Button('Cancel')],]

	window = sg.Window('Traffic Generator', layout4) #make the window

	event, traffic_values = window.read()

	
	if event == sg.WIN_CLOSED or event == 'Cancel': # if user closes window or clicks cancel
		window.close()
	elif event == 'Submit' or event == 'Confirm':
		window.close()
		
	traffic_to_trigger = list(filter(lambda x: traffic_values[x]==True, traffic_values.keys()))

	layout = [  [sg.Text('Enter the Network range with subnet(Eg - 192.168.1.1/24) : '), sg.InputText()], [sg.Button('Ok'), sg.Button('Cancel')] ]
	layout1 = [  [sg.Text('Enter the Server IP : '), sg.InputText()], [sg.Button('Ok'), sg.Button('Cancel')] ]
	layout2 = [  [sg.Text('Select the source for input values you like ')],[sg.Button('nmap scanning'),sg.Button('Config file')]  ]
	layout3 = [  [sg.Text('Enter the Test Durarion (minutes) : '), sg.InputText()], [sg.Button('Ok'), sg.Button('Cancel')] ]
	layout4 = [  [sg.Text('Enter the Test Monitor Durarion (minutes) : '), sg.InputText()], [sg.Button('Ok'), sg.Button('Cancel')] ]

	window = sg.Window('Total Traffic Durarion', layout3) #make the window
	event, traffic_duration = window.read()
	print(list(traffic_duration.values())[0])
	traffic_overall_duration = int((list(traffic_duration.values())[0]))*10
	window.close()
	
	window = sg.Window('Traffic Monitor Durarion', layout4) #make the window
	event, traffic_monitor_duration = window.read()
	print(list(traffic_monitor_duration.values())[0])
	traffic_monitor_duration = int((list(traffic_monitor_duration.values())[0]))*10
	window.close()
	
	window2 = sg.Window('Window Title' , layout2)
	event , values = window2.read()

	if event == 'nmap scanning':
		window2.close()
		window = sg.Window('Window Title', layout)
		event, network_range = window.read()
		if event == sg.WIN_CLOSED or event == 'Cancel': # if user closes window or clicks cancel
			window.close()
		elif event == 'Ok' or event == 'Confirm':
			window.close()
			print('Scanning for available devices in a network may take about 30 seconds.......')
			command = "nmap -sn " + network_range[0] + " | grep for | awk '{print $5}'"
			nmap_command = os.popen(command).read()
			available_ip = nmap_command.split('\n')
			print(available_ip)
			window1 = sg.Window('Window Title', layout1)
			event,server_ip = window1.read()

			if event == 'Ok':
				window1.close()
				#time.sleep(4)
				nmap_scanning_input()
			else:
				print('Terminating the process')
				window1.close()

	elif event == 'Config file':
		window2.close()
		window1 = sg.Window('Window Title', layout1)
		event,server_ip = window1.read()
		
		print(server_ip)
		server_ip = list(server_ip.values())[0]
		
		if event == 'Ok':
			window1.close()
			with open("config_dut.yaml",'r') as f:
				conf = yaml.safe_load(f) 
				config_file_input(conf)
				
		else :
			print('Terminating the process')
			window1.close()     

	elif event == sg.WIN_CLOSED:
		window2.close() 
