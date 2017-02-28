Hello,

I have some questions regarding the Deploy Keys and OAuth tokens, in the context of access management to repositories for external services.

# Context

- An external **service that needs to be able to clone a repository**. Example: configuration management tool (i.e. Ansible, Saltstack, Puppet, etc) that needs to `git clone` its configuration before applying it.
- This repository is a **private** repository on Github
- This repository has **git submodules** (also hosted on private Github repositories)

# The step by step story

## Read-only cloning from a service

Accessing a repository content as a service be done through a *Deploy Key* added to the project. And it works perfectly fine.

## The limitation of Deploy Keys

Adding a *Deploy Key* to a Github project works fine. But it's **not possible to add it to another, different project**, and fails with the error: `Key is already in use` (this behavior seems intended, more on that below).

## The case of private submodules

When you have a private repository to clone and this repository has other private repositories as submodules, things start to hurt.

- You expect to be able do it with the same deploy key, since it's the **same service** trying to do a clone of **related repositories** for **the same purpose**
- ... But it's not possible to have the same deploy key
- ... and using different deploy keys is not possible

**Why is it not possible to use different deploy keys?**

Every Github repository is accessed through the same `github.com` domain so it's not possible to differenciate it in the SSH configuration; and I honnestly think it's dirty to do things like [1].

## Workaround from git itself

It would be possible to use different deploy keys by changing the SSH command used by git between each submodule fetch to take a different private key with `-i identity_file`, either by using the `GIT_SSH_COMMAND` [2] or by setting the `core.sshCommand` [3] configuration.

But this is honnestly just complicated to do, let alone integrate it in existing tools that rely on being able to clone a repository and its submodules to do some automated actions.

## Official solution

As stated in link [4], the problem seems to be known and **the recommended solution is to use OAuth tokens**.

## On OAuth tokens

So, now that the official solution seems to be "use oauth tokens as user to clone repositories", let's take a look at it.

1. Go to the Organization's Settings
2. Declare an OAuth Application
3. Get the `client_id` and `client_secret`
4. Use this `client_id` and `client_secret` to call a PUT on the `/authorizations` endpoint to generate an oauth token
5. Use this token to clone the repositories

However, what is happening here is not really appropriate for services access:

- To call the `/authorizations` you have to provide an **existing user's username** in addition to the `client_id` and `client_secret`
- Thus, the OAuth token is tied to the user {him,her}self, which we don't want in the case of a service account. THe last thing we want is the token to be invalidated when the user leaves the organizations and the service to break.
- Plus, the available scopes do not seem to be precise enough: giving access to private repositories gives access not only to the user's private repositories but also to the private repositories he can see for organizations he/she belongs to (I probably missed some rules here that allow to restrict it)

# Conclusion

- It's not possible to use the same Deploy Key on multiple projects
- It's not possible or hard/ugly to use multiple Deploy Keys on a repository + its submodules
- => Deploy Key are out
- It's not reliable on the long term to use OAuth token since they're bound to a user and since their scope since to not be able to enforce enough limits

Does this problem seems relevant to you? If so, do you have any solution for this case of "I want a service to access repositories with private submodules in a safe and clean way"?

Solutions I see right now are:

- Allow the same Deploy Key to be used on multiple projects. I guess this will not be possible since it seems it's an intended behavior, but I'm still interested in the details of the rationale behind this decision (which is probably a good one)
- Allow OAuth tokens to be generated without an associated user, just as a service account that can access an Organization's projects

I would definitely like to avoid the "real Github account" with username/password, for obvious reasons.

Thanks for your time.

- [1] https://gist.github.com/jexchan/2351996
- [2] https://github.com/git/git/commit/3c8ede3ff312134e84d1b23a309cd7d2a7c98e9c
- [3] https://github.com/git/git/commit/39942766ab9bc738f49f93d4c8ea68ffbaadc305
- [4] https://github.com/blog/1270-easier-builds-and-deployments-using-git-over-https-and-oauth

# Follow ups / answers

## First Github reply

> Thanks for hanging in there! After speaking with the team about this, I'd like to share their input with you here. From your document, you said:
> 
>     Allow OAuth tokens to be generated without an associated user, just as a service account that can access an Organization's projects
>     I would definitely like to avoid the "real Github account" with username/password, for obvious reasons.
> 
> We're not 100% sure about the reasons you're referring to. Could you help us understand what those reasons are?
> 
> If your server needs to access multiple repositories, you can choose to create a new GitHub account and attach an SSH key that will be used exclusively for automation. Since this GitHub account won't be used by a human, it's called a machine user. You can then add the machine user as collaborator or add the machine user to a team with access to the repositories it needs to manipulate. Our team wrote more about this here:
> 
> https://developer.github.com/guides/managing-deploy-keys/#machine-users
> 
> Would such an approach using machine users work for you? If not, could you tell us more about why that wouldn't work for your use case? Any information there would help. Thanks!

## Reaction

> On Mon, Feb 27, 2017 at 5:56 PM, XYZ (GitHub Staff)
> <support@github.com> wrote:
> >> I would definitely like to avoid the "real Github account" with username/password, for obvious reasons.
>
> > We're not 100% sure about the reasons you're referring to. Could you help us understand what those reasons are?
> 
> Sorry, I shouldn't have made the assumption that it was obvious, my bad.
> 
>  I was refering to the reason for which I believe Deploy Key notion
> exist at all in Github and other software: having a "real" user
> account, even if it's only used by a machine, implies that the
> password for this user has to be remembered, stored (securely), and
> that if for whatever reason it is ever leaked (chances increased by
> the fact that we have to store it somewhere), it's much harder to
> renew it (considering the account has been compromised): we have to
> remove the user, create a new one, assign it to projects again,
> generate new key and assign it to the user. And the main pain in that
> is the 2 first steps : manually remove the user and create it (since
> bot account creations are forbidden according to the link you gave
> me).
> 
>> If your server needs to access multiple repositories, you can choose to create a new GitHub account and attach an SSH key that will be used exclusively for automation. Since this GitHub account won't be used by a human, it's called a machine user. You can then add the machine user as collaborator or add the machine user to a team with access to the repositories it needs to manipulate. Our team wrote more about this here:
>> https://developer.github.com/guides/managing-deploy-keys/#machine-users
>> Would such an approach using machine users work for you? If not, could you tell us more about why that wouldn't work for your use case? Any information there would help. Thanks Alexis!
> 
> 
> Well, that's what I itended to do if you didn't come up with another solution.
> It will do the job for sure, but I wish there was a cleaner solution
> to this, i.e. exactly what Deploy Keys are now in Github, but with the
> possibility to assign them to multiple projects.
> 
> If you are promoting the so called "machine-users" way, may I ask then
> what would be the purpose of the Deploy Keys if they are not intended
> to avoid the downside of machine-users I mentionned ?
> 
> All the best,
