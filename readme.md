# Pterodactyl API v1 - Connect to WebSocket
## In the config wings (/etc/pterodactyl/config.yml)

change the allowed origins to accept your IP (it will be visible to everyone)

>  '*' = all (potential security fail)
```yml
allowed_origins: [ '*' ]
```

Restart Wings
```bash
sudo systemctl restart wings
```

## On your JavaScript Project

Install dependencies
```bash
npm install --save colors node-fetch@2.6.1 websocket
```

Write this code
```JavaScript
const readline = require('readline');
const color = require("colors")
const fetch = require("node-fetch");
const WebSocketClient = require("websocket").client;

const config = {
    "panelUrl": "https://**************.com",
    "pterodactylUserApiKey": "********************************",
    "serverUUID": "6e9fe293"
};

// ReadLine for the commands
const rl = readline.createInterface({
    input: process.stdin,
    output: process.stdout,
    terminal: false
});

// Init 
const prod = () => {

    //Fetch Token and socket URL
    fetch(`${config.panelUrl}/api/client/servers/${config.serverUUID}/websocket`, {
        headers: {
            "Accept": "application/json",
            "Content-Type": "application/json",
            "Authorization": `Bearer ${config.pterodactylUserApiKey}`
        }
    }).then(res => res.json()).then(body => {
    
        const token = body.data.token;
        const socket = body.data.socket;
        const client = new WebSocketClient();
    
        client.on("connectFailed", function(error) {
            console.log("Connect Error: " + error.toString());
        });
    
        client.on("connect", async function(connection) {

            // Send Token auth on connection
            await connection.sendUTF(`{"event":"auth","args":["${token}"]}`)

            // request logs after 1 second
            setTimeout(() => { connection.sendUTF(`{"event":"send logs","args":[null]}`) }, 1000)
            
            connection.on("error", function(error) {
                console.log("Connection Error: " + error.toString());
            });
    
            // receiving messages
            connection.on("message", function(message) {

                // return != UTF-8
                if (message.type != "utf8") return;

                // Return stats (memory usage, ...)
                if (message.utf8Data.startsWith(`{"event":"stats"`)) return;
                
                // Logs Install output
                if (message.utf8Data.startsWith(`{"event":"install output"`)) return console.log(color.blue(`[Pterodactyl Daemon]: ${JSON.parse(message.utf8Data).args.toString()}`))
                
                // Logs console output
                if (message.utf8Data.startsWith(`{"event":"console output"`)) return console.log(JSON.parse(message.utf8Data).args.toString())
                
                // On token expiring, close connection and restart program
                if (message.utf8Data.startsWith(`{"event":"token expiring"`)) { connection.close(); prod(); return;}
                
                // Logs status message (starting, start, stoping, stop, ...)
                if (message.utf8Data.startsWith(`{"event":"status"`)) return console.log(color.yellow(`Server marked as ${JSON.parse(message.utf8Data).args.toString()}`))
                console.log(color.red(message))
            });
    
            // On command on console
            rl.on('line', function(line){

                // If command "close", close connection and exit program but NOT close the server
                if (line === 'close') { connection.close(); process.exit(0); }
                
                // if command "power", change the power according to the argument (power kill = kill the server, power start = start the server, ...) 
                if (line.startsWith('power')) return connection.sendUTF(`{"event":"set state","args":["${line.split(/ +/)[1]}"]}`)
                connection.sendUTF(`{"event":"send command","args":["${line}"]}`)
            })
        });

        //Connect to socket URL
        client.connect(`${socket}`);
    })
}

// First start
prod()
```