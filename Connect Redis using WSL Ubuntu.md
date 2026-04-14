- To Connect run the below commands
	- Create a tunnel to the jump server using ssh creds in the linux server using the command
	- ssh -f -i ./ipfwd-rsa -L 1817:10.21.4.36:6379 ipfwd@10.16.0.3 -N (used to create a tunnel from port localhost:1817 to 10.21.4.36:6379 using the ssh creds of ipfwd user ssh key)
	- export REDISCLI_AUTH=266cbea1-b29c-418f-922c-1765df346f37 (setup the redis cli auth password)
	- Connect to the redis using redis-cli -h 127.0.0.1 -p <port> (here 1817)

- To Disconnect run the below commands
	- lsof -iTCP:<port> -sTCP:LISTEN --> Get the PID from the output
	- kill <PID> --> kills the process