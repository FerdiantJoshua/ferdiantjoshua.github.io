# SSH, Tunneling, and Proxy

## Use SSH Port Tunneling to Allow Proxy Connection

1. Domainesia

    1. Create the tunnel on `localhost:9999`

        ```sh
        ssh username@remote_ssh_server -p 64000 -ND 9999 -T -f
        ```

        Details:
        - -p is SSH connection port (64000 is Domainesia's SSH port)
        - -N is to not execute remote command (useful for only just forwaring port)
        - -D is for `bind_address:port` binding
        - -T is to disable pseudo-tty allocation (useful as we're not creating interactive shell)
        - -f is to put the `ssh` into background after authenticates  
        (to stop the process: use `ps -ef | grep ssh` then `kill <PID>`)

    2. Set the browser or system proxy to:

        Manual configuration:
        - SOCKS Host: `localhost`
        - Port `9999`

## Author

Ferdiant Joshua Muis
