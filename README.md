# Actual budget setup notes

So I've started to use [Actual Budget](https://actualbudget.org/) to manage my personal and joint finances to see where money is going and be more conscious of how I consume and save money. After talking with a friend about envelope budgeting and originally thinking of making my own application with Enable Banking, I eventually settled on using Actual Budget because it cuts a lot of the hassle and gives some nice features.

These notes do not go through the creation of the run environment, because your mileage may vary. I thought that the best option for me is a DigitalOcean droplet, since I feel that there I can have near-complete ownership of my data and configure the server like I want. Some of you may have a home lab or use some other cloud provider. You do you.

These notes were originally written for a friend to help them setting up Actual budget, but I figured that these may be beneficial for others, too, and therefore I'm publishing this.

## Actual setup

These notes were written weeks after the setup — hopefully this is at least somewhat correct because setting this up was quite a bit of trial and error :)

I personally felt better using the Google OpenID login than a password. It was also nice to tinker with the Google OpenID.

* Actual Budget is probably worth running it on a subdomain (`actualbudget.example.com`) or its own domain (`actualbudget-example.com`). I was just reading today that `example.com/actual` may have issues (https://github.com/actualbudget/actual/issues/672)

* Create the `caddy` and `actual_data` directories on the server before running `docker compose up`. Also read through these notes first, especially the next bullet and its sub-bullets.

* With the default settings, if I remember correctly, the first OpenID user gets admin rights, and because I wanted to be security-conscious, I initially had (IIRC):

  * ```
    ACTUAL_USER_CREATION_MODE: "manual"
    ACTUAL_OPENID_ENFORCE: "false"  
    ```
  * And **only the Actual Budget container running without Caddy**, so it was only visible on the server's localhost. I then made an SSH tunnel to the server and opened the server's localhost through that in the browser and created my own user there.
  * Under the "Server online" button there's User Directory, where I verified my account's info and also created an account for my wife. The username should be the Google email address (I used my personal Google account here).
  * Once the first user has been created, you can restore the settings to how they are in the Docker Compose, and after that nobody else should be able to get in, as the OpenID setup in the docker-compose blocks all accounts that are not created manually in the User Directory first.

* I made a separate Google account for Google OpenID because apparently Google has at some point randomly banned Google accounts because of Google Cloud usage, and I don't really feel like getting the account that my entire life depends on banned 😂

* Then https://console.cloud.google.com/auth/overview

  * Create a project
  * From the left sidebar, "Clients", Create new, Web Application
  * Authorized JavaScript Origins: https://actualbudget.example.com
  * Authorized redirect URIs: https://actualbudget.example.com/openid/callback
  * Put the details you get into Docker Compose
  * It's a bit confusing that in Google project's "Audience" I only have my personal email listed and my wife isn't there at all, and somehow the Google login still works for my wife. After consulting ChatGPT it's probably due to the limited scope Actual Budget requests from Google (See [this](https://developers.google.com/identity/protocols/oauth2/production-readiness/overview?authuser=9&hl=en&utm_source=chatgpt.com#:~:text=Exception%3A%20If%20the%20app%20only%20requests%20basic%20identity%20scopes%20(openid%2C%20email%2C%20profile)%2C%20any%20user%20can%20access%20without%20being%20on%20the%20allowlist) and [this, I guess](https://github.com/actualbudget/actual/blob/master/packages/sync-server/src/accounts/openid.ts#L169)).

## Enable Banking

* Enable Banking needs to be enabled in Settings from Advanced Settings and Experimental Features

  * https://actualbudget.org/docs/advanced/bank-sync/enable-banking/
* Create an account at https://enablebanking.com/
* Start the Enable setup in Actual
* Add a new API application in Enable

  * env Production
  * redirect url https://actualbudget.example.com/enablebanking/auth_callback
* Fill in whatever Actual asks for
* Link Accounts for all users' accounts that you want to use. Using this setup for e.g. your SO requires the other person to enable linking in your Enable account. If the other person is not OK with this, I suggest creating a separate instance for them and setting up things there.
* Once an account is linked to Enable, you can link it in Actual. Unlinked (in Enable) accounts show up when triggering the linking in Actual but the integration completes succesfully only for accounts that have been linked in Enable.
* The start date is probably best set reasonably close, like the beginning of September, unless you want to label a poopload of stuff and mess around with getting the account balances to show correctly. The Enable integration doesn't fetch transactions that are older than the Starting Balance.

## Creating categories and CLI

* I deleted all categories except Starting Balances. No idea if that one could have been deleted too, but it's a bit of a special snowflake so I left it. You can also leave Income and just rename it manually if you like.

* The CLI (https://actualbudget.org/docs/api/cli/) can be used from your own computer. see the link for instuctions.

  * You get `ACTUAL_SESSION_TOKEN` by looking at the header in brower Dev Tools when, for example, refreshing the page.
  * If encryption is enabled for a budget file (see below), `ACTUAL_ENCRYPTION_PASSWORD` is the encryption key.
  * `ACTUAL_SYNC_ID` is found from the budget file's Advanced Settings.

```bash
mkdir -p ~/.config/actual
echo '{"serverUrl":"https://actualbudget.example.com"}' > ~/.config/actual/config.json
export ACTUAL_ENCRYPTION_PASSWORD="..."
export ACTUAL_SESSION_TOKEN="..."
export ACTUAL_SYNC_ID="..."
```

* I have encrypted budget files (enabled from Settings), so using the Actual Budget CLI was a bit more tedious and commands would fail on a semi-random basis.

  * ChatGPT came up with the idea that if you run it through a wrapper like this, it works, and after that it did work every time:

```bash
actual_e2ee() {
  command actual sync --clear >/dev/null || return
  command actual "$@"
}
```

After that you can, for example, download the transactions from your online bank as CSV, grind them through some AI magic, and tell it to output a script that you can run to create the categories and category groups in Actual through the CLI using that wrapper.





