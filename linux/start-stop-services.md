# Comando Linux para iniciar ou parar serviços

## Iniciar um seviço (Apache2 por exemplo)
```bash
sudo systemctl start apache2
```
## Parar um serviço (Apache2 por exemplo)
```bash
sudo systemctl stop apache2
```

## Reiniciar um serviço (Apache2 por exemplo)
```bash
sudo systemctl restart apache2
```

## Reiniciar todos os serviços (Apache2, Mysql e Firewall)

```bash
sudo systemctl restart apache2
sudo systemctl restart mysql
sudo systemctl restart ufw
```

## Status de um serviço (Apache2 por exemplo)
```bash
sudo systemctl status apache2
```

## Habilitar um serviço
```bash
sudo systemctl enable apache2
```

## Desabilitar um serviço
```bash
sudo systemctl disable apache2
```

## Reiniciar todos os serviços (Apache2, Mysql e Firewall)
```bash
sudo systemctl restart apache2
sudo systemctl restart mysql
sudo systemctl restart ufw
```


## Desabilitar um lista de serviços
```bash
sudo systemctl disable apache2 mysql RebbitMQ Redis
```

## Listar os serviços em execução
```bash
sudo systemctl list-units --type=service
```

