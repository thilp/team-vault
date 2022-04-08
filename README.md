# team-vault

Encrypted secrets (for small teams), and scripts to read and update them.

These scripts are very thin wrappers around [`age`][age],
which replaces GnuPG as a [simpler, better](#why-not-gpg-pass-or-gopass) encryption tool.

[age]: https://github.com/FiloSottile/age

## Use cases

### Join the team

For this operation, you need an already-enrolled colleague.

1. [Install `age`](https://github.com/FiloSottile/age#installation).
2. Run `./cmd/enroll-myself-in-team-vault`.
    - It creates a `.my-secret-identity` file that **must not be shared**.
        You can back it up (securely) though.
    - It displays your public key, but you don’t need to remember it.
    - It adds your public key to `teams.identities`, which lists everyone
        for whom the secrets in this vault are encrypted.
    - Please take the opportunity to open `team.identities` and remove
        the keys of people who have left the team.
3. Publish your changes (no need for a pull request yet):
    - `git add team.identities && git commit -m 'adding my public key' && git push origin YOUR_NAME`.
4. Ask **someone else** in the team to:
    1. Pull your changes: `git fetch && git checkout YOUR_NAME && git diff main`.
    2. Run `./cmd/reencrypt-all-team-secrets`.
    3. Commit, push, review, and merge.

You can optionally add `./cmd` to your path
(to avoid having to `cd` here before reading/writing secrets).
Depending on your shell, run:

- bash: `echo "export PATH=\"\$PATH:$(pwd)/cmd\"" >>~/.bash_profile`
- zsh (macOS): `echo "export PATH=\"\$PATH:$(pwd)/cmd\"" >>~/.zprofile`
- fish: `set -a fish_user_paths (pwd)/cmd`

### Decrypt a secret

Use **read-team-secret**:

```bash
$ ./cmd/read-team-secret secrets/some-file
```

### Create or update a secret

Use **upsert-team-secret**, which reads your cleartext value from stdin:

```bash
# Note the initial whitespace to avoid recording the secret in your shell history.
$  echo "secret!" | ./cmd/upsert-team-secret secrets/some-file
```

You can also simply provide the secret value after starting the command.
Enter the value (nothing will be printed) followed by a newline and Ctrl-D
to finish:

```bash
$ ./cmd/upsert-team-secret secrets/some-file
   # ← blank line waiting for your input
```

### Create a completely new vault

1. Copy the contents of this repository wherever you need: `cp -r cmd/ .gitignore README.md /your/repo`.
2. Move there: `cd /your/repo`.
3. Run `./cmd/enroll-myself-in-team-vault`.
4. Start encrypting things with `./cmd/upsert-team-secret`.
5. Commit and push all the changed files to your git server.

## FAQs

### Why is the private key not password-protected?

It wouldn’t provide meaningful security:

- https://github.com/FiloSottile/age#passphrase-protected-key-files
- https://news.ycombinator.com/item?id=29600122

### Why not rely on GitHub keys?

[githubkeys]: https://github.com/FiloSottile/age#encrypting-to-a-github-user

age has that [cute feature][githubkeys] (because it also supports SSH keys):

```bash
$ curl https://github.com/benjojo.keys | age -R - example.jpg >example.jpg.age
```

We could replace `team.identities` with a simpler list of GitHub usernames!

Unfortunately we rely on a GitHub Enterprise that requires an API token to
access the equivalent URL, making it much less practical.
And relying on GitHub.com instead would often mean dealing with confusing
hacker handles instead of clean, predictable enterprise usernames.

### About alternatives

#### Why not [gpg][], [pass][], or [gopass][]?

[gpg]: https://gnupg.org/
[pass]: https://www.passwordstore.org/
[gopass]: https://github.com/gopasspw/gopass

Both pass and gopass rely on GPG for encryption, and GPG [is][schneier]
[pretty][latacora] [bad][green].

[schneier]: https://www.schneier.com/blog/archives/2016/12/giving_up_on_pg.html
[latacora]: https://latacora.micro.blog/2019/07/16/the-pgp-problem.html#encrypting-files
[green]: https://blog.cryptographyengineering.com/2014/08/13/whats-matter-with-pgp/

Also, my `gpg --list-secret-keys` says I should be able to decrypt things,
but I can’t because (a pause of 3 seconds and some obscure error message about
files not found). And nobody on or off the internet seems to know why.
I tire of parleying with that unhelpful, complicated tool.

(gopass has tentative support for [age][], but it crashed immediately when I
tried to use it.
pass has [a fork that replaces gpg with age][passage], written by age’s creator,
but the size of its userbase is unclear and it doesn’t seem to add significant
features compared to age alone.)

[passage]: https://github.com/FiloSottile/passage

#### Why not [SOPS][]?

[sops]: https://github.com/mozilla/sops

It supports [age][]!
(It even says "It's recommended to use age over PGP, if possible.")
But it also asks you to provide the list of recipients with such a clumsy
interface (explicitly listing keys in a flag or an env var) that I’d have to
write a wrapper to make that UI usable in our CLI use-case; at which point I
can cut the middle-man and just wrap age directly.

Its value seems to be in the fact that it supports multiple (and so many) ways
to encrypt your secrets, especially for cloud deployments.
That’s simply a different use-case.

#### Why not [KeePassXC][]?

[keepassxc]: https://keepassxc.org/

You still need to securely share the password protecting the database.
And it’s not easy to 4-eyes review changes to that database.

#### Why not KMS or AWS Secrets Manager?

It replaces simple-if-tedious key distribution via git
(`cmd/enroll-myself-in-team-vault` + `cmd/reencrypt-all-team-secrets`)
with an AWS IAM setup that may be as tedious or even impossible to achieve.

Where I work, the exclusive reliance on AWS roles and the central provisioning
of these makes it impossible for us to have a KMS key usable only by members of
our team.
