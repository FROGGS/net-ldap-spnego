# NAME

Net::LDAP::SPNEGO - Net::LDAP support for ntlm/spnego authentication

# SYNOPSIS

```perl
use Net::LDAP::SPNEGO;
use Mojolicious::Lite;

my $SERVER = $ENV{AD_SERVER} // die "AD_SERVER env variable not set";

my %cCache;

app->secrets(['My secret passphrase here']);

hook before_dispatch => sub {
    my $c = shift;

    # once the user property is set, we are happy
    # and don't try to re-authenticate
    return if $c->session('user');

    my $cId = $c->tx->connection;
    my $cCache = $cCache{$cId} //= { status => 'init' };
    my $authorization = $c->req->headers->header('Authorization') // '';
    my ($AuthBase64) = ($authorization =~ /^NTLM\s(.+)$/);
    for ($AuthBase64 and $cCache->{status} =~ /^expect(Type\d)/){
        my $ldap = $cCache->{ldapObj}
           //= Net::LDAP::SPNEGO->new($SERVER,debug=>0);
        /^Type1/ && do {
            my $mesg = $ldap->bind_type1($AuthBase64);
            if ($mesg->{ntlm_type2_base64}){
                $c->res->headers->header(
                   'WWW-Authenticate' => 'NTLM '.$mesg->{ntlm_type2_base64}
                );
                $c->render(
                   text => 'Waiting for Type3 NTLM Token',
                   status => 401
                );
                $cCache->{status} = 'expectType3';
                return;
            }
            # lets try with a new connection
            $ldap->unbind;
            delete $cCache->{ldapObj};
        };
        /^Type3/ && do {
            my $mesg = $ldap->bind_type3($AuthBase64);
            if (my $user = $mesg->{ldap_user_entry}){
                $c->session('user',$user->{samaccountname});
                $c->session('name',$user->{displayname});
                my $groups = $ldap->get_ad_groups($user->{samaccountname});
                $c->session('groups',[ sort keys %$groups]);
            }
            $ldap->unbind;
            delete $cCache->{ldapObj};
        };
    }
    $c->res->headers->header( 'WWW-Authenticate' => 'NTLM' );
    $c->render( text => 'Waiting for Type 1 NTLM Token', status => 401 );
    $cCache->{status} = 'expectType1';
};

get '/' => 'index';

app->start;

__DATA__

@@ index.html.ep
<!DOCTYPE html>
<html>
<head>
<title>NTLM Auth Test</title>
</head>
<body>
    <h1>Hello <%= session 'name' %></h1>
    <div>Your account '<%= session 'user' %>'
       belongs to the following groups:</div>
    <ul>
    % for my $group (@{session 'groups' }) {
        <li>'<%= $group %>'</li>
    % }
    </ul>
</body>
</html>
```

# DESCRIPTION

**Net::LDAP::SPNEGO** provides the essential building blocks to implement NTLM SSO
for Windows clients to webservers. It can be used to proxy NTLM tokens
from the webbrowser to an active directory server using the SPNEGO protocol.

The dialog between browser and server in an NTLM authentication dialog looks
like this:

```
1: C  --> S  GET ...

2: C <--  S  401 Unauthorized
             WWW-Authenticate: NTLM

3: C  --> S  GET ...
             Authorization: NTLM <base64-encoded type-1-message>

4: C <--  S  401 Unauthorized
             WWW-Authenticate: NTLM <base64-encoded type-2-message>

5: C  --> S  GET ...
             Authorization: NTLM <base64-encoded type-3-message>

6: C <--  S  200 Ok
```

In contrast to modern web apis, the NTLM authentication dialog relies on a presistant
connection between browser and server to correlate steps 3 and 5 of the dialog.

The example above uses [Mojolicious::Lite](https://metacpan.org/pod/Mojolicious::Lite) but there is no inherent link to
that particular framework except that NTLM authentication relies on persistant
http connetions (keepalive) to linke the multi step authentication together.
In other words, a cgi implementation will not work, since the cgi process gets.
restarted with every request.

Windows will only engage in seamless NTLM negotiation with sites residing in the
local zone this may have to be configured in the Internet Settings dialog.

# METHODS

**Net::LDAP::SPNEGO** provides all the methods of [Net::LDAP](https://metacpan.org/pod/Net::LDAP) as well as the following:

## my $response = $ldap->bind\_type1($type1B64)

Start binding the ldap connection. The argument to this method is the base64 encoded type1
NTLM token received from a browser request in the _Authorization_ header.

```
Authorization: NTLM base64encodedntlmtoken
```

The bind\_type1 call encodes this token in an SPNEGO message and uses it to
initiate a bind call to the active directory server.

The bind\_type1 call will return a [Net::LDAP::Message](https://metacpan.org/pod/Net::LDAP::Message) object received from the
AD server in the same way the [Net::LDAP](https://metacpan.org/pod/Net::LDAP) call will in a regular bind request.
If the request has been successful the response has an _ntlm\_type2\_base64_
property you can hand to your webbrowser for the next step.

```
WWW-Authenticate: NTLM $res->{ntlm_type2_base64}
```

## my $mesg = $ldap->bind\_type3($type3B64)

Complete binding the ldap connection. The argument to this method is the base64
encoded type3 NTLM token received from a browser request in the _Authorization_
header.

```
Authorization: NTLM base64encodedntlmtoken
```

The bind\_type3 call will return a [Net::LDAP::Message](https://metacpan.org/pod/Net::LDAP::Message) object received from the
AD server in the same way the [Net::LDAP](https://metacpan.org/pod/Net::LDAP) call will in a regular bind request.

The response object comes with an extra property: _ldap\_user\_entry_
containing the ldap user entry information.

```perl
{
  'pwdlastset' => '131153165937657397',
  'objectcategory' => 'CN=Person,CN=Schema,CN=Configuration,DC=oetiker,DC=local',
  'displayname' => 'Tobi Test',
  'usncreated' => '362412',
  'distinguishedname' => 'CN=TobiTest TT. Tobi,CN=Users,DC=oetiker,DC=local',
  'countrycode' => '0',
  'whenchanged' => '20160820154613.0Z',
  'instancetype' => '4',
  'lastlogontimestamp' => '131161815735975291',
  ...
}
```

## my $group\_hash = $ldap->get\_value\_ad\_groups($username)

Query the ldap server for all the users group memberships,
including the primary group and all the inherited memberships due to
a group being a member of another group.

The function uses the magic _member:1.2.840.113556.1.4.1941:_ query
to effect a recursive search.

The function returns a hash indexed by the _sAMAccountName_s of the groups
containing the DN and the description of each group.

```perl
{
 'Remote Desktop Users' => {
   'dn' => 'CN=Remote Desktop Users,CN=Builtin,DC=oetiker,DC=local',
   'description' => 'Members in this group are granted the right ...'
  },
 'Users' => {
   'dn' => 'CN=Users,CN=Builtin,DC=oetiker,DC=local',
   'description' => 'Users are prevented from making accidental ...'
 },
 'Domain Users' => {
   'description' => 'All domain users',
   'dn' => 'CN=Domain Users,CN=Users,DC=oetiker,DC=local'
  }
}
```

# EXAMPLE

The included example script `eg/mojolite-demo.pl` shows how to use the module to implement
NTLM authentication for a [Mojolicious::Lite](https://metacpan.org/pod/Mojolicious::Lite) webapplication. Use the following steps
to run:

```
$ perl Makefile.PL
$ make 3rd
$ env AD_SERVER=ad-server.example.com ./eg/mojolite-demo.pl deamon
```

Now connect with your webbrowser to the webserver runing on port 3000. If you
login from a Windows host and the url you are connecting resides in the local zone,
you will see (or rather not see) seemless authentication take place. Finally
a webpage will be displayed showing a list of groups you are a member of.

The demo script stores your authentication in a cookie in your brower, so once
you are authenticated, you will have to restart the browser or remove the cookie
to force another authentication.

# ACKNOWLEGEMENTS

Implementing this module would not have been possible without the access
to these imensly enlightening documents:
[NTLM Authentication Scheme for HTTP](http://www.innovation.ch/personal/ronald/ntlm.html) by Ronald Tschalär,
[The NTLM Authentication Protocol and Security Support Provider](http://davenport.sourceforge.net/ntlm.html) by Eric Glass
as well as [The PyAuthenNTLM2 Module](https://github.com/Legrandin/PyAuthenNTLM2) by Helder Eijs.

Thank you for makeing that information avaialble.

# COPYRIGHT

Copyright (c) 2016 by OETIKER+PARTNER AG. All rights reserved.

# LICENSE

This program is free software; you can redistribute it and/or modify it under the same terms as Perl itself.

# AUTHOR

Tobias Oetiker <tobi@oetiker.ch>

# HISTORY

```
2016-08-19 to 0.1.0 initial version
```
