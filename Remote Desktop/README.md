# Remote Desktop

## Initall xfce4 and xrdp

`sudo apt install xfce4 -y`

Choose `lightdm` when prompted during the installation

`sudo apt install xrdp -y`

`echo xfce4-session > ~/.xsession`

`sudo service xrdp restart`

`ifconfig | grep inet`

Take note of the inet address

## Access using Windows RDP

It is now to try to access via `Windows Remote Desktop Connection`
