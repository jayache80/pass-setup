# Setting up pass
This document contains the following guides:
1. How to set up a new password store
1. How to migrate a local password store to a central server
1. How to set up an existing password store on another client

## How to set up new a password store

#### Create GPG keys

```
gpg --gen-key
```

Follow the prompts. Eventually, it will output something like:

```
public and secret key created and signed.

pub   rsa2048 2020-09-30 [SC] [expires: 2022-09-30]
      38294DB32940D3892AB90843Q40398J3V30J207T
      uid                       Full Name <your.email@domain.com>
      sub   rsa2048 2020-09-30 [E] [expires: 2022-09-30]
```

In the above output, your "gpg-id" is your email address on the `uid` line, and
we will use this gpg-id in the next step to initialize your password store.

#### Initialize pass

```
pass init your.email@domain.com
```

You can now add existing passwords, randomly generate new passwords, and
retrieve stored passwords that are saved as GPG-encrypted files in
`~/.password-store` by default. If you have no hopes of ever migrating this
password store to another computer, you can stop now; `pass` will be storing
your encrypted files as `.gpg` files within `~/.password-store`. However,
setting up a git repository is nice because it will keep a `git log` of the
changes made to the password store; the `pass` utility automatically makes `git
commits` to the repository any time you change the password store in any way.
Further, using a git repository allows you to easily migrate the password store
to other computers.

#### Setup git repo
> Note: Make sure you have `[user]` info in `~/.gitconfig`.
Simply run:
```
pass git init
```

This will work great if the passwords are never meant to leave this computer.
But what if you want another computer to have access to this password store? In
that case, you should migrate the git repostory that `pass` created for you to
a central server.

## How to migrate a local password store to a central server
The `pass` utility has set up the password store git repository for you, which
by default lives in `~/.passwore-store`. To migrate this to a central server,
log in to the central server and create a bare repository with:
```
git init --bare /path/to/repo.git
```
where `repo` is how you'd like to name this password store.

Set up SSH public/private keys and make this server passwordless.

Move back to the machine where you initially created the password store so that
we can push it to this bare repo. For `git` to be able to sync with a remote
SSH repo on a different port, put the following in `~/.ssh/config`:
```
Host central
    Hostname <centralhost>
    User <user>
    IdentityFile ~/path/to/central_id_rsa
```

Then, on the machine where you initially create the password store, run:
```
pass git push --mirror ssh://central:<port>/path/to/repo.git
```

## How to set up an existing password store on another client

### Export/import GPG keys
On a machine that has the existing GPG keys used on an existing pass git repo,
identify the private key of interest by running
```
gpg --list-secret-keys
```

Using the `uid` that you found out from the above command, export the key to
`other` host by running:
```
gpg --export-secret-keys <uid> | ssh user@other "gpg --import --no-tty --batch --yes"
```

> Note: I found it best to set up `~/.gnupg/gpg-agent.conf` with the following
> line:
>
> `pinentry-program /path/to/a/gui/pinentry`
>
> This way, when you run the above command, SSH will use your terminal to
> prompt you for a password to `other` host, while the GUI pinentry will prompt
> for the password using a separate window. This prevents the confusing
> scenario of a TTY pinentry and SSH password prompt stepping on each other.
> Alternatively, set up your SSH connection to be passwordless (use a
> public/private key) so that it won't prompt for the password.

After exporting the GPG key to the `other` host, you should be able to see it
by running on the `other` host:
```
gpg --list-secret-keys
```

You may notice that the `uid` field shows `[unknown]` beside it, where on the
source system it showed `[ultimate]`. This is because it's not trusted yet.

### Trust GPG key
```
$ gpg --edit-key <uid>
Command> trust
...
# Choose 5, I trust ultimately
Your decision? 5
Do you really want to set this key to ultimate trust? (y/N) y
```

### Synchronize the password store with remote git repo
If we were creating a fresh password store, we would've run `pass init <uid>`
with the `uid` we learned of earlier. However, a git repo exists somewhere that
already has the `uid` baked into it. We've migrated over the GPG key, now we
just need to clone the git repo. Running `pass init <uid>` usually creates a
`~/.passwore-store` directory for us. Since we're not running that, we'll have
`git clone` implicitly create the directory for us.

For `git` to be able to sync with a remote SSH repo on a different port, put
the following in `~/.ssh/config`:
```
Host central
    Hostname <centralhost>
    User <user>
    IdentityFile ~/path/to/central_id_rsa
```

This assumes you've already set up public/private keys to log into the
`central` server, that the public key (`central_id_rsa.pub`) has already been
copied over to the `central` server, and that this machine has a copy of the
private key (`central_id_rsa`) located at `~/path/to/central_id_rsa`.

> Note: Make sure you have `[user]` info in `~/.gitconfig`.

If this were the first time we were creating a git repo for this password
store, we would've ran `pass git init` but in this case we want to clone an
existing git repo into `~/.passwore-store`:
```
git clone ssh://central:<port>/path/to/repo.git ~/.passwore-store
```

You should now be ready to rock and roll.

