# Prise en main de Hashicorp Vault
```bash
$role_name=xxx
$role_id=xxx
$secret_id=xxx

# Création de l'app_role my-role
./vault write auth/approle/role/my-role1 secret_id_ttl=24h token_num_uses=100 token_ttl=24h token_max_ttl=24h secret_id_num_uses=100

# Attachement d'une policy à un approle
vault write auth/approle/role/my-role token_policies="<app_role>"

# S'authentifier avec un app_role
./vault write auth/approle/login role_id=$role_id secret_id=$secret_id

# Récupérer le secret_id d'un app_role
./vault write -f auth/approle/role/$role_name/secret-id
./vault write -force auth/approle/role/$role_name/secret-id

# Lister les app-roles
vault list auth/approle/role/

./vault write -f auth/approle/role/my-role/secret-id

# Récupérer role-id
./vault read auth/approle/role/my-role/role-id

# Lister les policies
./vault policy list

# Vérifier l'attachement de la policy
./vault write auth/approle/login role_id=$role_id secret_id=$secret_id
```


# Connexion au Vault de test sur le <server_name>
```bash
cd /appli/projects/PEX/vault_indus/
export VAULT_ADDR='http://127.0.0.1:8002'
export VAULT_TOKEN="<root_token>"
```

## Mise en place d'une politique permettant d'accéder au secret du point de montage
```
path "automation_stack1/*" { capabilities = [ "create", "read", "update", "delete", "list" ] }
```
