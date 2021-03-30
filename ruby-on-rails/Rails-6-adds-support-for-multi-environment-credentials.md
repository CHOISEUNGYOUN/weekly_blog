## Rails 6 adds support for multi environment credentials

### Background
Generally in applications there are various secrets and credentials, that we need to make use of like API keys, secrets, etc. For such secrets we need the ability to conveniently and securely manage credentials.

Rails 5.1 added [a feature](https://github.com/rails/rails/pull/28038) to use `secrets` to manage credentials.

Rails 5.2 replaced `secrets` with `credentials`, since encrypted and un-encrypted secrets were making it harder to manage them.

A set of files were used to manage these credentials:

* `config/credentials.yml.enc`
* `config/master.key`
  
`config/credentials.yml.enc` is an encrypted file which store the credentials. As this is a encrypted file, we can safely commit it to our version control systems.

`config/master.key` contains `RAILS_MASTER_KEY` which is used to decrypt the `config/credentials.yml.enc`. We should not commit this file to version control.

### Interacting with credentials
As `config/credentials.yml.enc` is encrypted we should never directly read from or write to it. Instead, we will use utilities provided by Rails which abstract encryption and decryption process for us.

### How to add/update credentials?
We can edit the `credentials` by running the following command:

```shell
$ EDITOR=vim rails credentials:edit
```

This will open a vim editor with the decrypted version of the credentials file.

We can add new credentials in YAML format. Lets add the following lines, save the changes, and exit.

```yaml
aws:
  access_key_id: 123
  secret_access_key: 345
github:
  app_id: 123
  app_secret: 345
secret_key_base:
```

When we save it, it encrypts again using the same master key.

If default editor is not set and we haven’t specified the editor, then we get the following message:

```shell
$ rails credentials:edit
No $EDITOR to open file in. Assign one like this:

EDITOR="mate --wait" bin/rails credentials:edit

For editors that fork and exit immediately, it's important to pass a wait flag,
otherwise the credentials will be saved immediately with no chance to edit.
```

### How to read credentials?
We can now access the credentials in the following way:

```rb
> Rails.application.credentials.config
#=> {:aws=>{:access_key_id=>"123", :secret_access_key=>"345"}, :github=>{:app_id=>"123", :app_secret=>"345"}}
> Rails.application.credentials.github
#=> {:app_id=>"123", :app_secret=>"345"}
> Rails.application.credentials.github[:app_id]
#=> "123"
```
### Managing multi environment credentials before Rails 6
There was no built in support for multiple environment credentials before Rails 6. We could manage credentials for different environments but it was upto us to explicitly specify which set of credentials to use for a specific environment.

We could store the credentials in a single file as below:

```yaml
development:
  aws:
    access_key_id: 123
    secret_access_key: 345

production:
  aws:
    access_key_id: 1f3649fe-ebbd-11e9-81b4-2a2ae2dbcce4
    secret_access_key: 203060d3a5456fa6cd2da3c958001440    
```
Then, config can be accessed using the following command:

```rb
> Rails.application.credentials[Rails.env.to_sym][:aws][:access_key_id]
#=> "123"
```
### There are some problems with this approach:
* There is just 1 master key and everyone on the development team had access to it. Which means everyone on the development team had access to production environment.
* We needed to explicitly specify which environment credentials to use in the code.

Another way to manage environment specific credentials was by creating environment specific files. For example, we can create `config/staging.yml.enc` for staging environment and `config/production.yml.enc` for production environment. To read config from these files, Rails 5.2 [provided](https://github.com/rails/rails/commit/68479d09ba6bbd583055672eb70518c1586ae534) `encrypted` method to support for managing multiple `credentials` files.

This approach involved writing even more boiler plate code to manage the keys and the encrypted files for every environment.

### In Rails 6
Now, Rails 6 [has added](https://github.com/rails/rails/pull/33521) support for multi environment credentials.

It provides utility to easily create and use environment specific credentials. Each of these have their own encryption keys.

### Global Credentials
The changes added in the above PR are backwards compatible. If environment specific credentials are not present then rails will use the global credentials and master key which are represented by following files:

* `config/credentials.yml.enc`
* `config/master.key`

We use the global configuration only for `development` and `test` environments. We share the `config/master.key` with our entire team.

### Create credentials for production environment
To create credentials for `production` environment, we can run the following command:

```shell
$ rails credentials:edit --environment production
```

The above command does the following:

* Creates `config/credentials/production.key` if missing. Don’t commit this file to VCS.
* Creates `config/credentials/production.yml.enc` if missing. Commit this file to VCS.
* Decrypts and opens the production credentials file in the default editor.

We share the `production.key` with limited members of our team who have access for production deployment.

Let’s add following credentials and save:

```yaml
aws:
  access_key_id: 1f3649fe-ebbd-11e9-81b4-2a2ae2dbcce4
  secret_access_key: 203060d3a5456fa6cd2da3c958001440
```

Similarly we can create credentials for different environment like `staging`.

### Using the credentials in Rails
For any environment Rails automatically detects which set of credential to use. Environment specific credentials will take precedence over global credentials. If environment specific credentials are present, they will be used else Rails will default to global credentials.

For development:

```rb
$ rails c
> Rails.application.credentials.config
#=> {:aws=>{:access_key_id=>"123", :secret_access_key=>"345"} }}
> Rails.application.credentials.aws[:access_key_id]
#=> "123"
```

For production:

```rb
$ RAILS_ENV=production rails c
> Rails.application.credentials.config
#=> {:aws=>{:access_key_id=>"1f3649fe-ebbd-11e9-81b4-2a2ae2dbcce4", :secret_access_key=>"203060d3a5456fa6cd2da3c958001440"}}
> Rails.application.credentials.aws[:access_key_id]
#=> "1f3649fe-ebbd-11e9-81b4-2a2ae2dbcce4"
```

### Storing encryption key in environment variables
We can also set the value of the encryption key in specific environment variable Rails will auto detect and use it.

We can either use the generic environment variable `RAILS_MASTER_KEY` or an environment specific environment variable like `RAILS_PRODUCTION_KEY`

If these variable are set then we don’t need to create the `*.key` files. Rails will auto detect these variables and use them to encrypt/decrypt the credential files.

The environment variable can be used for example on `Heroku` or similar platforms.

```shell
# Setting master key on Heroku 
heroku config:set RAILS_MASTER_KEY=`cat config/credentials/production.key`
```

Original Source:
[Rails 6 adds support for multi environment credentials](https://blog.saeloun.com/2019/10/10/rails-6-adds-support-for-multi-environment-credentials)
