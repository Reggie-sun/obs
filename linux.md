查找所有包含 todesk 的 systemd 服务 
systemctl list-unit-files | grep -i todesk 
或：systemctl | grep -i todesk

如果你想卸载
sudo find / -name "uninstall" 2>/dev/null | grep  

列出服务
systemctl  list-units --type=service | grep
systemctl --user list-units --type=service | grep