If you are like me with multiple avatars, `foo` and `bar` on the GitHub, and want to access GitHub with both those avatars from the same machine. Here is how I do it and her is you too could do it.

First I would create two differet keys to access each of those users.

```
ssh-keygen -b 4096 -C foo@gmail.com
ssh-keygen -t ecdsa -b 521 -C bar@hotmail.com
```

Second, I create two folders, `foo-gh` and `bar-gh`. I would usually create them in my home directory.

```
mdkir -p ~/foo-gh
mkdir -p ~/bar-gh
```

Third I create `.gitconfig` files in each of those folder, `~/foo-gh/.gitconfig.foo`...

```
$ ~/foo-gh/.gitconfig.foo
[user]
email = mojozoox@gotmail.com
name = Foo Fighters

[github]
user = "foo"

[core]
sshCommand = "ssh -i ~/.ssh/id_rsa"
```

... and `~/bar-gh/.gitconfig.bar`

```
$ ~/bar-gh/.gitconfig.bar
[user]
email = mojozoox@gotmail.com
name = Foo Fighters

[github]
user = "bar"

[core]
sshCommand = "ssh -i ~/.ssh/id_ecdsa"
```

Finally I create `~/.gitconfig`

```
[includeIf "gitdir:~/foo-gh/"] # include for all .git projects under ~/foo-gh/
        path = ~/foo-gh/.gitconfig.foo

[includeIf "gitdir:~/bar/"] # include for all .git projects under ~/bar-gh/
        path = ~/bar/.gitconfig.bar
```

Now I could use all the `foo`'s repositories from `~/foo-gh` and `bar`'s repositories from `~/bar-gh`.
