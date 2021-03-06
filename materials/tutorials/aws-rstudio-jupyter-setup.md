# Setting up and customizing Louis Haslett's Rstudio AMI

For running Rstudio server on an AWS machine, plus additional customizations that allow us to run raster-vision, Jupyter notebooks, etc. 

# Install Rstudio-server AMI

The links to these can be found [here](http://www.louisaslett.com/RStudio_AMI/).

## Configure 
Allow HTTP access and ssh access from relevant addresses, give it a 60GB
EBS volume

## Check that python3 is installed
This is an Ubuntu 16.04 install, with python 3.5 as the latest

# Install docker on Ubuntu

Following these instructions: https://www.digitalocean.com/community/tutorials/how-to-install-and-use-docker-on-ubuntu-16-04

# Installing Anaconda/Jupyter notebooks on Rstudio-server Ubuntu
I followed these [directions](https://medium.com/@alexjsanchez/python-3-notebooks-on-aws-ec2-in-15-mostly-easy-steps-2ec5e662c6c6), starting at step 7 because we already have our AWS machine setup. I followed the instructions exactly, except at Step 12, I made one small change from instructions. I changed this line in the text I edited in `jupyter_notebook_config.py` from: 

```vim
c.NotebookApp.certfile = u'/home/ec2-user/certs/mycert.pem'
```

To:
```vim
c.NotebookApp.certfile = u'/home/ubuntu/certs/mycert.pem'
```
Because we are running on an Ubuntu machine rather than Amazon Linux. I also had to copy in the sha key that I set up (which the directions state explicitly). 

## Add permissions to allow access to port 8888

Before being able to successfully launch a notebook from my local machine, I had to open access to port 8888 for the IP addresses that the instances will be used from, adding a custom TCP rule to the Inbound section of the instance's security group. 

## Running the Jupyter notebook

While ssh'd into the rstudio instance, you can simply run:
```bash
jupyter notebook
```
And then copy the public DNS from the instance into the browser address bar, such that it looks like this: 

https://ec2-000-11-222-333.compute-1.amazonaws.com:8888/

Note that you have to get the correct DNS address from the EC2 console.

Follow the instruction to enter the browser with an untrusted certificate. 

You will be prompted to enter the password we set up (ask me if you don't have it), and then you will be in. 

# Running the raster-vision-examples jupyter notebook

This is a bit different, because you are asked to start jupyter via their script:

```bash
scripts/jupyter
```
I had to do a few things to be able to get to run the notebook they way they ask you to.

First, I had to make user ubuntu capable of executing docker commands:
```bash
sudo usermod -a -G docker ubuntu
```

Then I had to go into the script itself:
```bash
vim scripts/jupyter
```

And change all to 8080 settings to 8888, which is the port jupyter notebooks use in the setup we just created. It now looks like this. 

```bash
docker run ${RUNTIME} ${NAME} --rm -it ${TENSORBOARD} ${AWS} ${RV_CONFIG} \
    -v "$SRC"/spacenet:/opt/src/spacenet \
    -v "$SRC"/notebooks:/opt/notebooks \
    -v "$SRC"/data:/opt/data \
    -p 8888:8888 \
    ${IMAGE} \
    /run_jupyter.sh \
    --ip 0.0.0.0 \
    --port 8888 \
    --no-browser \
    --allow-root \
    --notebook-dir=/opt/notebooks;
```

That actually allowed me to log into the notebook, with one last step.  When loading the notebook as the instructions say:

```bash
scripts/jupyter
```

You can get an output that looks so:
```bash
    Copy/paste this URL into your browser when you connect for the first time,
    to login with a token:
        http://(3bd8e26b6bfa or 127.0.0.1):8888/?token=<a long token string>
```
Obviously the token looks different. To get the jupyter notebook in your local browser, you have to append that token to the url we use to get the notebook:
```bash
https://ec2-000-11-222-333.compute-1.amazonaws.com:8888/?token=<a long token string>
```

And that gets you into the notebook. I had to add one final step to be able to read the datasets out of the spacenet s3 bucket, which was to attach an AmazonS3ReadOnlyAccess policy to the user whose policy credentials are being used for the instance. 

# Getting jupyter to run from Rstudio-server terminal
This required a few more gymnastics. The default user for Rstudio is rstudio.  That's where you get logged in when you enter Rstudio-server through the browser. I got jupyter set-up to run from /home/ubuntu. I could change into the user ubuntu from Rstudio's terminal, using:

```bash
sudo -u ubuntu bash
```

But scripts/jupyter would fail because jupyter was always trying to write temp files to /home/rstudio, and didn't have permissions. 

So I added ubuntu as an Rstudio user.  From root:

```bash
sudo echo 'server-user=ubuntu' >> /etc/rstudio/rserver.conf
sudo rm -rf /var/log/rstudio-server
sudo rm -rf /var/lib/rstudio-server
sudo rm -rf /tmp/rstudio-rsession
sudo rm -rf /tmp/rstudio-rserver
sudo rstudio-server restart
ps aux | grep ubuntu
```

Had to change the password for ubuntu to get this to work:
```bash
sudo passwd ubuntu
```

After doing that and another `rstudio-server restart`, I was able to log in to rstudio with the user ubuntu and the new PW. 

And scripts/jupyter now works from the rstudio terminal. 




