# Validator Migration

Migration process of validator service is important part of providing infrastructure services for Vidulum chain.

When validator needs to be migrated?
Here is few situations which might required validation service to be moved:
 - Server ran out of resources and can't be expanded
 - System crashed and needs to be reinstalled
 - Maintenance of current system and validation service will be moved to backup node for that period

::: tip NOTE:
Please follow migration steps very carefuly. If you are not sure about certain steps or need more clarification, reach out to our validator community on Discord, please.
Be asware that running two validators with same key will lead to slashing and might lead to validator account being tombstoned.
:::



## SECTION 1: Preparation
Configuration of node and validator is stored in
```bash
${HOME}/.vidulum/config
```

When you look into that folder you will find 

```bash
.vidulum/
└── config
    ├── addrbook.json
    ├── app.toml
    ├── client.toml
    ├── config.toml
    ├── genesis.json
    ├── node_key.json
    └── priv_validator_key.json
```



## SECTION 2: Migration steps

### Existing Validator

::: tip NOTE:
All steps and references to folders assume that node/validator configuration matches Vidulum deployment guides. 
In case you have different names for services and folders adjust command accordingly, please.
:::

Login to existing validator 


First stop service on existing system
```bash
sudo systemctl stop vidulum.service
```

Once service is stopped rename exist