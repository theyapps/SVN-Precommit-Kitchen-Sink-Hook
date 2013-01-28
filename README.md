# NAME

pre-commit-kitchensink-hook.pl

# SYNOPSIS

    pre-commit-kitchen-sink-hook.pl [-file <ctrlFile>] \\
	[-fileloc <cntrlFile>] (-r<revision>|-t<transaction>) \\
	[-parse] [-svnlook <svnlookCmd>] [<repository>]

    pre-commit-kitchen-sink-hook.pl -help

    pre-commit-kitchen-sink-hook.pl -options

# DESCRIPTION

This is a Subversion pre-commit hook that can check for several issues
at once:

- This hook can verify that a particular user has permission to change a
file. The file can be specified as either a _glob_ or _regex_ format.
Even better, you can also specify an _ADD-ONLY_ setting that allows you
to create tags via an `svn copy`, but won't allow you to modify the
files in that directory. This protects tags from being modified by a
user.
- This hook can verify that a particular property has a particular value
and has been set on a particular file. You can specify the files via
_regex_ or _glob_ format, and you can specity the value of the
property either via a _string_, _regex_, or _numeric_ value.
- This hook can prevent the user from adding files with banned names. For
example, in Windows you cannot have a file that starts with  `aux` or
`prn` or `con`. You also can't have file names that have `:`, `^`,
and several other types of characters in them. You may also want to ban
file names that have spaces in them since these tend to cause problems
with some utilities. This only affects newly added or renamed files, and
not current files.
- This hook can also verify that particular revision properties are set.
However, this only works on Subversion release 1.5 or greater. But,
since Subversion release 1.4 is no longer supported, you really
shouldn't be using releases older than 1.5 anyway.

This hook works through a control file that is in standard Windows
Inifile format (described below). This allows you to set permissions and
other changes without having to modify this program itself.

# OPTIONS

## Control File Definition

The _Control File_ controls the way this hook operates. It can control
the permissions you're granting to various users, your group
definitions, what names are not allowed in your repository, and the
properties associated with the file and revision.

There are two places where the control file can be stored. The first is
inside the Subversion repository server as a physical text file. This
keeps the control file away from prying eyes. Unfortunately, it means
that you must have login access to the repository server in order to
maintain the file.

The other place is inside the repository itself. This makes it easy to
maintain. Plus, since it's in a Subversion repository, you'll see who
changed this file, when, and why. That can be nice for auditing.
Unfortunately, this file will be visible to everyone. If you have an
LDAP password in that file, you'll be exposing that password to all of
your users.

You are allowed to use either a physcial control file, a control file
stored in the repository, or both. You must have at least one or the
other defined.

Format of the control file is discussed below. See ["CONTROL FILE LAYOUT"](#CONTROL FILE LAYOUT)

- \-file

    The location of the physical text control file stored on the Subversion
    repository server. Normally, this is kept in the `hooks` directory
    under the Subversion repository directory.

- \-fileloc

    The location fo the control file inside the Subversion repository. This
    should use the `svnlook` format. For exmaple, if you have a directory
    called `control` under the root of your repository, and inside, your
    control file is called `control.ini`, the value of this parameter would
    be `/control/control.ini`.

## Other Options

- \-r

    The Subversion repository revision. Normally, this is only used for
    testing purposes since you really want the transaction number of the
    commit and not the revision number. Good for testing. This parameter
    cannot be used at the same time as the `-t` parameter.

- \-t

    The Subversion repository Transaction Number. This is passed to the
    scrip pre-commit found in the Repository's hook directory as a
    parameter. You need to modify the pre-commit script to pass the
    transaction number to this script.

- \-svnlook

    This is the full path to the svnlook command. The full path is needed
    because for security reasons, the `PATH` environment variable is empty
    when the hook is executed. Default is /usr/bin/svnlook.

- \-parse

    Used mainly for debugging. This will dump out the entire configuration file
    structure, so you can verify your work. It will also test the control file
    and let you know if there are any errors.

- \-help

    Prints a helpful message showing the different parameters used in
    running this pre-commit hook script.

- \-options

    Prints a helpful message showing a detailed explanation of  the
    different parameters used in running this pre-commit hook script.

- <repository>

    The location of the Subversion repository's physical directory. Default
    is the parent directory.

# CONTROL FILE LAYOUT

The hook is controlled by a user defined control file. The control is in
Windows' IniFile layout. This layout consists of _Comments_, _Section
Headings_, and _Parameter Lines_.

## Comments

Comment lines begin with either a `#`, or a `;` or a `'`. Blank lines
are also ignored. All other lines must either be a Parameter Line or a
Section Heading.

## Basic Layout

The basic layout is a Section Heading followed by a bunch of Parameter
Lines.

Section headings are enclosed in square brackets. The first word
in a section heading is the type of section it is (Group, File,
Property, Revprop, or Ban). The rest is a description of that section
heading. For example:

    [File Users may not modify a tag.]

In the above example, the section is _File_ and the description is
_Users may not modify a tag._. 

Unlike in ["IniFile" in Config](http://search.cpan.org/perldoc?Config#IniFile), the descriptions don't have to be unique.
However, the descriptions are used in user error messages, so be sure
to put in a good description that is user friendly. For example:

    [File Only approved users may modify the Control File]

is a better section heading than:

    [File Read-only on Control.properties]

Under each section is a series of Parameter lines that apply to that section.

Parameter Lines are in the form of

    <Key> = <Value>

where _Key_ is the parameter key, and _Value_ is the value for that
parameter. Notice that the two are separated by an equal sign. Spaces
around the equal sign are optional, and the _Value_ is the entire line
including spaces on the end of the line, so be careful.

The layouts of the various sections and their permitted parameters are
explained below. Note that section type names and parameter keys are
case insensitive.

### Group

This is used to define user groups which can be used to define file
permissions. This makes it easier to keep track of file permissions as
people move from project to project. You only have to change the group
definition.

The section layout is thus:

    [GROUP <GroupName>]
    users = <ListOfUsers>

The `GroupName` is the name of the group. Group names should be
composed of just letters, numbers, and underscores, and should contain
no white space. Group names are case insensitive.

User names in this hook substitute an underscore for a space and ignore case
For example, if the user signs into Subversion as _john doe_, their user name
will become _JOHN\_DOE_. User names cannot start with an _at sign_ (@).

The `ListOfUsers` is either a whitespace or comma separated list of user
names. This list can also contain the names of other groups. However, the
groups are calculated from the top of the control file to the bottom, so
if group "A" is contained inside group "B", you must define group "A" before
you define group "B".

    [GROUP developers]
    users = larry, moe, curly

    [group admins]
    users = tom, dick, harry

    [group foo]
    users = @developers, @admins, @bar, alice

    [group bar]
    users = bob, ted, carol

In the above example, everyone in group _DEVELOPERS_ and group _ADMINS_
is in group _FOO_. However, members of group _BAR_ won't be included.

### File

A file Section Heading starts with the word `file` and an explanation
of that section. It then consists of the following parameters:

- match

    A Perl style expanded regular expression matching the files that are
    affected by this permission definition. Note this cannot be used with
    the `file` parameter.

    If the `match` string contains the text <USER>, this text
    will be substituted by the name of the user from the `svnlook author`
    command. This is done to allow you to create special user directories or
    files that only that user can modify. For example:

        [FILE Users can only modify their own watchfiles]
        match = ^/watchfiles/.*
        access = read-only
        users = @ALL

        [FILE Allow user to edit their own watchfiles]
        match = ^/watchfiles/<USER>\.cfg$
        access = read-write
        users = @ALL

    The above will prevent users from modifying each other watch files, but
    will allow them to modify their own watch files.

    __Word o' Warning__: This script normalizes user ID to all caps. This
    means that if the user name is _bob_, it will become _BOB_. Thus,
    _<USER>_ also becomes `BOB`.



- file

    An Ant style  globbing expression matching the files that are affected
    by this permission definition. This cannot be used with the `match`
    parameter.

    Regular expressions are much more flexible, but most people are not too
    familiar with them. In a file glob expression, an asterisk (\*)
    represents any number of characters inside of a directory. Double
    asterisk (\*\*) are used to represent any number of directory levels. A
    single quesion mark (?) represents any character. All glob expressions
    are anchored to the front of the line and the back of the line.

    Therefore:

        file = *.jpg

    does not mean any file that end with `jpg`, but only files in the root
    of the repository that end with `jpg`. To mean all files, you need to
    unanchor the glob expression as thus:

        file = **/*.jpg

    Notice too that directory names will end with a final slash which is
    a great way to distinguish between a file being added or deleted
    and a directory being added or deleted. For example:

        file = /tags/*/

    will refer only to directories that are subdirectories directly under the
    tags directory, and not to files. You'll see this is a great way to protect
    your tagged versions from being modified when [access](http://search.cpan.org/perldoc?access) is discussed below.

    If the `file` string contains the text `<USER>`, this text
    will be substituted by the name of the user from the `svnlook author`
    command. This is done to allow you to create special user directories or
    files that only that user can modify. For example:

        [FILE Users can only modify their own watchfiles]
        file = /watchfiles/**
        access = read-only
        users = @ALL

        [FILE Allow user to edit their own watchfiles]
        file = /watchfiles/<USER>.cfg
        access = read-write
        users = @ALL

    The above will prevent users from modifying each other watch files, but
    will allow them to modify their own watch files.

- users

    A list of all users who are affected by this type of access. Groups can
    be used in a user list if preceeded by an _at sign_ (@). There is one
    special group called `@ALL` that represents all users.

- access

    The access permission on that file. Notice that there is no permission
    for preventing a user from seeing the contents of the file, only for
    changing the file. This is because the trigger is on the `commit`
    and not on `checkout`. If you need to prevent people from seeing
    the file, you must do this with your repository access.

    Access is determined in a top down fashion in the control file. The first
    entry might take a particular user's ability to commit changes to a particular
    file, but a section further down might allow that same user the ability
    to commit changes to that same file. While the third might remove it again.

    Access is granted or denied by the __last__ matching access grant.

    - read-write

        Files marked as `read-write` means that the user may commit changes to
        the file. They can add a new file or directory by that name, delete it, or
        modify it.

    - read-only

        Files marked as `read-only` means that the user cannot commit changes to
        that file. The user is not allowed to create, delete, or modify this
        file.

    - no-delete

        Files marked as `no-delete` allow the user to modify and even create the
        file, but not delete the file. This is good for a file that needs to be
        modified, but you don't want to be removed.

    - no-add

        Files marked as `no-add` allow the user to modify the file if it already
        exists, and are allowed to delete a file if it already exists, but they
        are not allowed to add a new file to the repository. This is combined
        with the [no-delete](http://search.cpan.org/perldoc?no-delete) option above to allow people to modify files, but
        not to add new files or delete old ones from the repository.

    - add-only

        This is a special access permission that will allow a user to add, but not
        modify a file with that matching pattern or regex. This is mainly used to 
        ensure that tags may be created but not modified. For example, if you have
        a `/tags` directory, you can do this:

            [FILE Users cannot modify any tags]
            file = /tags/**
            access = read-only
            user = @ALL

            [FILE Users can add tags to the tag directory]
            file = /tags/*/
            access = add-only
            user = @ALL

        The first [File](http://search.cpan.org/perldoc?File) section removes the user's ability to make any changes in
        the `/tags` directory. The second [File](http://search.cpan.org/perldoc?File) section allows users to only add
        directories directly under the `/tags` directory. Thus, a user can do
        something like this:

            $ svn cp svn://localhost/trunk svn://localhost/tags/V-1.3

        to tag version 1.3 of the source, but then can't modify, add, or delete any
        files under the `/tags/V-1.3` directory. Also, users cannot do something
        like this:

            $ svn cp svn://localhost/trunk svn://localhost/tags/V-1.3/BOGUS

        because the ability to copy a directory is only allowed in the `/tags/`
        directory and no subdirectory.

        Nor, can a user do this:

            $ touch tags/foo.txt
            $ svn add tags/foo.txt
            $ svn commit -m"Adding a file to the /tags directory"

        Since only directories can be added to the `/tags` directory.

- case

    This is an optional parameter and its only valid value is _ignore_. This
    allows you to ignore case when looking at file names. For example:

        [file Bob can edit our batch files]
        file = **/*.bat
        access = read-write
        users = bob
        case = ignore

    This will match every Batch file with a suffix of _\*.bat_ or _\*.BAT_ or
    even _\*.Bat_ and _\*.BaT_.

### Property

This section allows you to set what properties and what those property
values must be on a particular set of files. The section heading looks
like this:

    [property <purpose>]

Where `purpose` is the purpose of that property. This is used as an
error message if a file fails to have a required property or that
property is set to the wrong value. The following parameters are used:

- match

    A Perl style expanded regular expression matching the files that are
    affected by this property. Note this cannot be used with the `file`
    parameter.

- file

    A glob type expression matching the files that are affected by this
    property. Note this cannot be used with the `match` parameter.

- case

    An optional parameter and the only permitted value is _ignore_. This
    means to ignore the case of file names (not property values! They're
    still case sensitive)

- property

    The name of the property that should be on this file. 

- value

    The value the property should have. Note that this can be either a
    string, a number of a regular expression that the value should match.
    This is determined by the _type_ parameter.

- type

    The type of value that the _value_ parameter actually contains. This
    can be `string`, `number`, or `regex`.

### Revprop

This section allows you to set what revision properties and what
those property values must be when committing a revision.

    [revprop <purpose>]

Where `purpose` is the purpose of that property. This is used as an
error message if the revision fails to have a required revision property
or that property is set to the wrong value.

The Revision property of `svn:log` is a special revision property
that all Subversion clients and server versions can use. This is
set by the `-m` parameter in the `svn commit` command. This can
be used to verify that the commit log message is in the correct format.

Here is an example:

    [revprop Users must have at least 10 characters of a commit message]
    property = svn:log
    value = ^.{10,}
    type = REGEX

The above requires a user to create an at least 10 character commit
message when committing a change. This will prevent a user from leaving
a null message. You can get even more complex and imaginative:

    [revprop Users must include a Jira ticket number with a commit, or NONE]
    property = svn:log
    value = ^(NONE)|(([A-Z]{3,4}-\d+(,\s+)?)+):.{10,}
    type = REGEX

Imagine you have a ticketing system where ticket numbers start with a three
or four capital letter issue type, followed by a dash, followed by an
issue number (much like Jira). The above will require a user to list
one or more comma separated issue IDs, followed by a colon and a space. If
there is no issue ID, the user can use the word "NONE". This has to
be followed by at least a ten character description. The following
would be valid commit comments:

    NONE: Fixed the indentation of a few files.
    FOO-123: Added a new format to match the specs
    BAR-334, BAZ-349: Fixed the display bug

While the following wouldn't be allowed:

    Fixed stuff
    AAAA
    FOO-123: SSDd

If both the client and server run a release of Subversion 1.5 or
greater, you can use other revision properties on a commit.

Revision properties are set with the `--with-revprop` on the
`svn commit` command. Revision properties other than `svn:log`
can only be used when the Subversion client and server revisions
are 1.5 or newer.

__WARNING:__ Users with a Subversion client older than 1.5
won't be allowed to commit changes if revision properties other
than `svn:log` are required.

- property

    The name of the revision property that should be set in this revision.

- value

    The value the revprop should have. Note that this can be either a
    string, a number of a regular expression that the value should match.
    This is determined by the _type_ parameter.

- type

    The type of value that the _value_ parameter actually contains. This
    can be `string`, `number`, or `regex`.

### Ban

    This section allows you to ban certain file names when you add a new
    file. For example, in Unix, a file can be called C<aux.java>, but
    such a name would be illegal in Windows. 

    The section heading looks like this:

    [ban <description>]

    where E<lt>descriptionE<gt> is the description of that ban. This is
    returned to the user when a banned name is detected.

- match

    A Perl style regular expression matching the banned names. Note this
    cannot be used with the `file` parameter.

- file

    An Ant style globbing expression that matches the banned names. Note
    this cannot be used with the `item` parameter.

- case

    This is an optional parameter. If set to _ignore_, it will ignore the
    case in file names.

### Ldap

This section defines an LDAP server for the purposes of pulling of LDAP
groups to use in this hook. For example, if you have a Windows Active
Directory server, you could use your Windows groups to put users into
groups for this particular hook.

Due to the complexities of the LDAP query and the wide variety used by
various groups, it is somewhat necessary to play loose and fast with the
way attributes work.

The section heading will look like this:

    [ldap <ldapURI or server>]

Where the name of the LDAP server (or full URI can be contained in the
section name.

- ldap:xxxxx

    This allows you to further define how to do ldap binding. The "xxxx" is
    the option in the Net::LDAP constructor. For example, `ldap:scheme` will
    be the _scheme_ option in the Net::LDAP constructor. The most common ones
    are:

    - ldap:password

        The password for the ldap server

    - ldap:scheme

        This could be `ldap` or `ldaps`

    - ldap:port

        The port number for LDAP.

- bind

    The _Distinguished Name_ used by the LDAP server for _binding_
    (logging in) to the ldap server. If not given, anonymous binding will be
    done.

- bind:xxxx

    This is an option used in the `bind` method in the [Net::LDAP](http://search.cpan.org/perldoc?Net::LDAP) class.
    The most common ones are:

    - bind:password

        The password for the Distinguish name.

- filter

    This is the attribute you are filtering on via the Ldap name of the
    user. In Windows Active Directory, this is usually something like
    `sAMAccountName`.

    This is used by the _search_ method of the Net::LDAP module. For
    example, let's say that you've filtered on `sAMAccountName` for Ldap
    user `johnsmith`. The call to the Net::LDAP search method will
    look something like this:

        my $search = $ldap->search(filter => "sAMAccountName=johnsmith");

- search:xxxxxx

    Other attributes to use for the search method. Common ones are

    - search:attrs

        A list of attributes to return.

- groupAttr

    The attribute in the member LDAP record that contains the group names.
        It is normally `memberOF` in Window's ActiveDirectory.

- regex

    When LDAP returns a group, it returns the full _Common Name_ for that
    group. Normally, you only want the first part of the name. For example,
    the LDAP _CN_ might be:

        CN=AppTeam,OU=Groups,OU=Accounts,DC=mycorp,DC=com

    but you want just the _AppTeam_ as the group name. You can use this
    to pinpoint the group name in the expression. You use parentheses to
    mark the place in the regular expression where you will find the
    name of the group. For example, the following will pinpoint the
    name of the group (_AppTeam_) in the above expression:

    	regex = ^[^=]+=([^,]+)

    Note that the quoting slashes are not around the regular expression.
    __DO NOT INCLUDE THEM!__. Otherwise, your group anames will be messed up.

    In fact, the above regular expression is pretty much the stanadard one
    that you will use.

    Note that the group name will be _normalized_ which means that it will
    be uppercased and blanks will be replaced by underscores.

#### LDAP Example

Here's an example of what an LDAP section might look like. It pretty
much follows what someone who uses Windows Active Directory for
corporate access. 

	[ldap ldap://ldapserver.mycorp.com:389]
	bind = CN=subversion,OU=Users,OU=Accounts,DC=mycorp,DC=com
	bind:password = Sw0rdfi$h
	filter = sAMAccountName
	groupAttr = memberOf
	regex = ^CN=([^,]+),

# AUTHOR

David Weintraub
[mailto:david@weintraub.name](mailto:david@weintraub.name)

# COPYRIGHT

Copyright (c) 2010 by David Weintraub. All rights reserved. This
program is covered by the open source BMAB license.

The BMAB (Buy me a beer) license allows you to use all code for whatever
reason you want with these three caveats:

1. If you make any modifications in the code, please consider sending them
to me, so I can put them into my code.
2. Give me attribution and credit on this program.
3. If you're in town, buy me a beer. Or, a cup of coffee which is what I'd
prefer. Or, if you're feeling really spendthrify, you can buy me lunch.
I promise to eat with my mouth closed and to use a napkin instead of my
sleeves.
