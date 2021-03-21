# Test - Deploy to private VMs from AZ DevOps

## Links / Resources

<https://docs.microsoft.com/en-us/learn/modules/host-build-agent/4-create-build-agent?source=learn>

<https://docs.microsoft.com/en-us/azure/devops/pipelines/tasks/deploy/ssh?view=azure-devops>

<https://app.diagrams.net/#G1UhI7-1chSGqp6XJtIODUlhy9GXIU4t55>

## Create a new Organisation to house

Go to http://dev.azure.com

Create a new organisation

Create a new project (called "BuildAgent")

> Note: Instructions are followed using WSL2:Ubuntu on local machine... cloud shell will probably work too.

## Create a Resource Group for easy deletion

Group name: `DevOpsBuildAgentTest`

## Create VMs

Public server (Build Agent)

```sh
az vm create \
    --name MyLinuxAgent \
    --resource-group DevOpsBuildAgentTest \
    --image Canonical:UbuntuServer:18.04-LTS:latest \
    --location eastus \
    --size Standard_DS2_v2 \
    --admin-username azureuser \
    --generate-ssh-keys
```

Private server

```sh
az vm create \
    --name MyPrivateHost \
    --resource-group DevOpsBuildAgentTest \
    --image Canonical:UbuntuServer:18.04-LTS:latest \
    --location eastus \
    --size Standard_DS2_v2 \
    --admin-username azureuser \
    --generate-ssh-keys \
    --public-ip-address ""
```

## Create agent pool

See: <https://docs.microsoft.com/en-us/learn/modules/host-build-agent/4-create-build-agent?source=learn>

Using the same settings where possible for simplicity

(Create and save a Personal Access Token as per instructions too)

## Add SSH private key to the `MyLinuxAgent` VM

Get the IP address of the build server

```sh
IPADDRESS_AGENT=$(az vm show \
 --name MyLinuxAgent \
 --resource-group DevOpsBuildAgentTest \
 --show-details \
 --query [publicIps] \
 --output tsv)
```

SSH to the server to confirm it's working

```sh
ssh azureuser@$IPADDRESS_AGENT
```

Get the IP address of the private server

```sh
IPADDRESS_PRIVATE=$(az vm show \
 --name MyPrivateHost \
 --resource-group DevOpsBuildAgentTest \
 --show-details \
 --query [privateIps] \
 --output tsv)
```

SSH to the server to confirm you can't (it's on a different subnet)

```sh
ssh azureuser@$IPADDRESS_PRIVATE
```

## Install the latest build agent on the agent box

See: <https://docs.microsoft.com/en-us/learn/modules/host-build-agent/4-create-build-agent?source=learn>

```sh
curl https://raw.githubusercontent.com/MicrosoftDocs/mslearn-azure-pipelines-build-agent/master/build-agent.sh > build-agent.sh
```

Replace anything that doesn't look right with the actual value

```sh
export AZP_AGENT_NAME=MyLinuxAgent
export AZP_URL=https://dev.azure.com/TC-DevOpsLearning
export AZP_TOKEN=<TOKEN>
export AZP_POOL=MyAgentPool
sudo apt update && sudo apt install -y jq
export AZP_AGENT_VERSION=$(curl -s https://api.github.com/repos/microsoft/azure-pipelines-agent/releases | jq -r '.[0].tag_name' | cut -d "v" -f 2)
```

Install the agent (assuming you trust the sh script)

```sh
chmod u+x build-agent.sh
sudo -E ./build-agent.sh
```

## Install Docker engine on the private server

1. Copy the private key from `~/.ssh/id_rsa`
2. SSH to the build agent server `ssh azureuser@$IPADDRESS_AGENT`
3. Paste the private key in to `~/.ssh/id_rsa` on the private server
4. Set correct settings `chmod 0600 ~/.ssh/id_rsa`
5. SSH to private server from build agent `ssh 10.0.0.5` (whatever the private IP is)

> The rest of these steps are from <https://docs.docker.com/engine/install/ubuntu/>

6. Ensure old Docker installs don't exist `sudo apt-get remove docker docker-engine docker.io containerd runc`
7. Update apt `sudo apt update`
8. Install apt https packages

   ```sh
   sudo apt-get install -y \
   apt-transport-https \
   ca-certificates \
   curl \
   gnupg \
   lsb-release
   ```

9. Add docker GPG key `curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg`
10. More steps!
    ```sh
    echo \
    "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
    $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
    ```
11. Install Docker

    ```sh
    sudo apt-get update
    sudo apt-get install -y docker-ce docker-ce-cli containerd.io
    ```

12. Check it's working: `sudo docker run hello-world`

## Create an SSH service connection for the agent to use (In Az DevOps)

Values

- Host Name: `10.0.0.5`
- Port: `22`
- Private Key: (COPY KEY FROM `~/.ssh/id_rsa` file)
- Username: `azureuser`
- Service connection name: `ssh-MyPrivateHost-service-connection`

## Create a pipeline that creates a docker container
