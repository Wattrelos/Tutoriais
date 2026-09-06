
# Configurar Firewall UFW

## Configurar o Firewall (se UFW estiver ativo)

```bash
ufw allow 'Apache'
ufw status
```

- Isso libera a porta 80 (HTTP) para acesso externo. 

## Gerenciar o serviço

- Reiniciar: systemctl restart apache2.
- Parar: systemctl stop apache2.
- Iniciar: systemctl start apache2.
Habilitar no boot: systemctl enable apache2. 