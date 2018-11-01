# Spatial CIMIS

For 2018 Spatial CIMIS will begin processing the next generation GOES-16/17
data.  This will require new infrastructure (new dish and servers) and a slightly
modified spatial CIMIS toolset to process the new spatial data.


# Development CIMIS Processor

These instructions are for setting up the spatial CIMIS program for either
Ubuntu (UCD) or Red Hat / Fedora (DWR) based servers.

Spatial CIMIS is run primarily with the GRASS GIS program.  However,
there are some additional steps that need to take place. 


## Install GRASS7

### GrassModules

### Grass database, ~/gdb

## Install Spatial CIMIS

```
sudo su - cimis
git clone -b GOES-16-17 https://github.com/CSTARS/spatial-cimis
ln –sf spatial-cimis/gdb
grass -text ~/gdb/cimis/PERMANENT
```

The cimis user will need the ET APP key.   `~cimis/.grass7/rc` should contain:

```
MAPSET: 500m 
ET_APPKEY: {your app key} 
GISDBASE: /home/cimis/gdb 
LOCATION_NAME: cimis 
GUI: text 
```
Verify ET_APPKEY by running GRASS and checking with the `v.in.et -?` command:
```
GRASS 7.4.0 (cimis):~ > v.in.et -?
GRASS 7.4.0 (cimis):~ > g.gisenv
```

### Install and Configure Incron

With incron installed ensure the cimis user can add to its incrontab file:

`echo cimis >> /etc/incron.allow`

### Setup GOES.mk

The following incron job copies cloud cover data into the Grass DB (GOES16 or GOES17). 

Edit the goes.mk file and update the following line to ensure it points to the correct
home directory.
```
sudo su – cimis 
vi ~/spatial-cimis/g.cimis/etc/goes.mk 
```

Make sure this line looks like this:
```
files:=$(wildcard /home/cimis/CA/*.pgm) 
```
```incrontab –e  
/home/cimis/CA IN_MOVED_TO \ 
  grass /home/cimis/gdb/goes16/cimis \ 
  --exec /home/cimis/spatial-cimis/g.cimis/etc/goes.mk \ 
  --directory=/home/cimis/spatial-cimis/g.cimis/etc files=$@/$# import solar 
```

### Solar Calculation

The clear sky solar calculation uses the cloud cover data from the GOES16/17 
Grass DB to calculate the actual solar net radiation.  Currently runs at 
the end of the day (not real time as of yet) and takes about **25 minutes for each day**. 

```
grass solar/cimis 
```
always start in the cimis mapset to retain bash history
```cd solar 
g.mapset 20180810 
g.list rast  
```
Lists solar calculations; raster results with –G are finished.  The ssetr –G is used by the cimis program 
```
make –directory=~/spatial-cimis/g.cimis/etc/ -f solar.mk solar {-n} 
```
The `-n` option does a check, doesn't actually execute.
```
g.mapset 20180811 
g.list type=rast pattern=ssetr* 
```
Lists ssetr –G –Gc –Gi –K rasters 

Check to see which dates have incomplete solar calculations 
```
for m in  201808??;do x=`g.list type=rast pattern=ssetr-G mapset=$m`;echo $m  $x;done 
```
This determined that 20180812 was the last day with calculations.  Must start on the 13th. 
```
g.mapset 20180813
```
The make command looks at all cloud cover datasets then runs heliosat command.  
Run at the end of the day. (no real time calculation currently). 
```
make --directory=~/spatial-cimis/g.cimis/etc/ -f solar.mk solar 
```

```
for m in 201810??;do echo $m;  
  g.mapset $m; 
  make --directory=~/spatial-cimis/g.cimis/etc/ -f solar.mk solar;
done 
```

### ETo Calculation
Once the solar calculation for the day is complete run the final ETo calculation.
```
sudo su - cimis
grass cimis/cimis 
cd cimis 
g.mapset 20180813 -c 
make --directory=~/spatial-cimis/g.cimis/etc --file=cimis.mk ETo 
```

# GOESBOX 

## Software Configuration (Open Suse Leap 42.3 or newer)

### Incron

Incron setup for grb-box.cstars.ucdavis.edu or DWR receiver.  
[Download RPM package](https://software.opensuse.org/package/incron?search_term=incron) 
for OpenSuse Leap 42.3 and install.

`rpm -i /root/incron-0.5.10-2.1.x86_64.rpm`

Register startup script:  `insserv incron`.  
More details at http://inotify.aiken.cz/?section=incron&page=download&lang=en

Create the cimis user and CA subset directory 
```
useradd –m –c “Spatial CIMIS” cimis
mkdir /grb/raw/CA ; chown cimis /grb/raw/CA
cd ; ln -sf /grb/raw/CA
```
Subset data should reside on the largest storage array 

Clone repo 

```
su - cimis ; git clone -b GOES-16-17 https://github.com/CSTARS/spatial-cimis
```

Pre-setup incron 
```
echo cimis >> /etc/incron.allow 
$grb-box=$(find cd /home/cimis/spatial-cimis/grb-box |grep –v README) 
sudo cp -v $grb-box /usr/local/grb-box 
sudo chmod 644 *.mk 
sudo chmod 754 goestcl 
chgrp cimis goesctl *.mk 

```

On the grb-box create a passwordless ssh key pair for the rsync exchange and add public key to `cimis@GOESBOX/.ssh/authorized_keys`.  Pre-pend string `from="IP of GRB Box receiver"` to limit access only from the GRB-BOX. 

```
ssh-keygen -t rsa -b 4096 -C "cimis@grb-box.cstars.ucdavis.edu"

incrontab –e 
/grb/raw/fulldisk IN_CREATE /usr/local/grb-box/goesctl raw=$@/$# CA 
/home/cimis/CA IN_CREATE /usr/local/grb-box/goesctl ca=$@/$# push 
```

Initially connect with the rsync ssh connection with the key pairs to accept the connection.  The incrontab session should now be pushing CA cropped band 2 images to the cimis server. 

Keep the last 6 months of CA subsetted data on the grb-box 

```crontab -e 
4 4 * * * find /home/cimis/CA/ -mtime +180  -name \*.pgm | xargs rm 
```

# Final Configuration

## Pushing GOES16 Data

### UCD GOES16 data replaces DWR GOES15 data
As of October 23rd 2018 the production DWR and UCD Spatial CIMIS processors no longer processes GOES15 data from their respective on site dishes and receivers.  Instead they pull daily GOES16 CIMIS data products from the UCD GOES16 processor and use existing processes to continue to provide raster outputs, Arc/Info ASCII Grid output as well as ETo data for the webAPI.

Each GOES15 processor run scripts as the `cimis` user to accomplish this.

3:15am `/home/cimis/bin/cg.tunnel` 

Pulls daily GOES16 grass maps to `/home/cimis/gdb/cimis`

3:30am `/home/cimis/bin/cg.grass.script /home/cimis/bin/cg.daily.output`

The `cg.daily.output` grass script runs a modified version of the raster output grass makefile `png-special` which creates the daily raster images and zipcodes.

Scripts available in this [repo](https://github.com/CSTARS/spatial-cimis/tree/GOES-16-17/bin).

