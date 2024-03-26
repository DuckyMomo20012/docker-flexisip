# Setup flexisip Proxy Server Authentication

> [!NOTE]
> The demo use the domain `nip.io` to simulate the domain configuration. You can
> see the documentation for more information: [nip.io](https://nip.io/).

The demo configuration for file `flexisip.conf` is defined with the following
content:

```conf
[global]
aliases=192.168.1.17.nip.io
[module::Authentication]
enabled=true
auth-domains=account.192.168.1.17.nip.io
available-algorithms=SHA-256
db-implementation=soci
soci-backend=mysql
soci-connection-string=db=flexisip user=mysql password=mysql host=db.192.168.1.17.nip.io
soci-password-request=select password, algorithm from passwords join accounts on passwords.id = accounts.id where accounts.username = :id and accounts.domain = :domain
[module::Registrar]
reg-domains=account.192.168.1.17.nip.io
```

- `reg-domains`: with the default value `localhost`, which means the `flexisip`
  server will only accept `REGISTER` requests from the domain `localhost`. See
  more from [Check service ports](#check-service-ports).
  - This value SHOULD match the `APP_SIP_DOMAIN` env defined in `flexiapi` from
    the [`ubuntu23-10/.env.flexiapi`](./ubuntu23-10/.env.flexiapi) file.

  - Without enabling the `Authentication` module, you can use Linphone or any
    other SIP client to register many account as you want with the domain
    `localhost`.

- `auth-domains`: list of domain to check credentials for the `REGISTER`
  requests. You can use wildcard `*` to accept all domains, or stricter to only
  check the domain that match `reg-domains`.

- `available-algorithms`: list of available algorithms for the password
  encryption. The `flexiapi` server hash the password with the algorithm
  `SHA-256`, this can be change using the `ACCOUNT_DEFAULT_PASSWORD_ALGORITHM`
  env from the [`ubuntu23-10/.env.flexiapi`](./ubuntu23-10/.env.flexiapi) file.

> [!NOTE]
> Zoiper 5 only support `MD5` algorithm, so you have to switch
> `available-algorithms` to `MD5` in the `flexisip.conf` file. You can check
> the documentation for more information: [Setup
> Softphone](./docs/setup-softphone.md).

> [!CAUTION]
> DO NOT mix the hash algorithm in the `available-algorithms` like `MD5
> SHA-256`, so the `flexisip` server can register requests from the Linphone and
> Zoiper 5.



- `db-implementation`: the database implementation to store the account
  information. We use `soci` to read the account information from the MariaDB.

- `soci-backend`: the database backend to connect to the MariaDB.

- `soci-connection-string`: the connection string to connect to the MariaDB.

  - Note that the `flexisip` server **cannot lookup the domain from container service
    name**.

- `soci-password-request`: the SQL query to get the password and algorithm for
  the account.

  - The `:id` and `:domain` are the placeholders for the account username and
    domain.

  - The query should return the password and algorithm for the account.

  - The `flexisip` server will hash the password with the algorithm before
    comparing it with the `REGISTER` request.

      - With the algorithm `SHA-256`, with `username` is `bob`, `domain` is
        `localhost` and `password` is `password`, the hashed password will be
        hashed as follows:

        ```text
        HASH = SHA-256("bob:localhost:password") = 8be01ad9dc03ef2a610bf13c7e307e78dd72c973ad365eb2b128505f8dbeaa74
        ```

        Check with terminal:

        ```bash
        echo -n "bob:localhost:password" | sha256sum
        # 8be01ad9dc03ef2a610bf13c7e307e78dd72c973ad365eb2b128505f8dbeaa74
        ```

      - You can check the documentation the [SIP authentication
        process](./docs/sip-auth-process.md) for the example packet capture and
        the authentication process.

> [!NOTE]
> You can check these configuration documentation in file
> [`ubuntu23-10/flexisip_conf/flexisip.conf.default`](./ubuntu23-10/flexisip_conf/flexisip.conf.default)
> for more information.
