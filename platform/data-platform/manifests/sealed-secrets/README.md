Aquí van (generados por `P9/scripts/03-seal-platform-secrets.sh`):

- `sealed-postgresql.yaml`  -> Secret `sa-platform-postgresql`  (postgres-password, password)
- `sealed-rabbitmq.yaml`    -> Secret `sa-platform-rabbitmq`    (rabbitmq-password, rabbitmq-erlang-cookie)

Son SealedSecret (cifrados): es seguro versionarlos. Sin ellos, PostgreSQL y
RabbitMQ no arrancan (usan `existingSecret`).
