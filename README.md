# NAME

Template::Provider::PerContextDBIC - Load templates using DBIx::Class with per-context resultsets

# VERSION

version 0.000001

# SYNOPSIS

    use My::DBIC::Schema;
    use Template;
    use Template::Provider::PerContextDBIC;

    my $schema = My::DBIC::Schema->connect(
        $dsn, $user, $password, \%options
    );
    my $resultset = $schema->resultset('Template');
    
    my $dbic_provider = Template::Provider::PerContextDBIC->new({
                RESULTSET => $resultset,
                TOLERANT => 1,
                # Other template options like COMPILE_EXT...
            });

    my $template = Template->new({
        LOAD_TEMPLATES => [ $dbic_provider ],
    });

    # Process the template 'my_template' from resultset 'Template'.
    $template->process('my_template');
    # Process the template 'other_template' from resultset 'Template'.
    $template->process('other_template');

If you have a resultset that changes based on context, update `resultset` 
between calls to `process`.

    # Process the template 'my_template' for site 'foo' from resultset 'Template'.
    $dbic_provider->resultset(
        $schema->resultset('Template')->search({site=>'foo'})
    );
    $template->process('my_template');

    # Process the template 'other_template' for site 'bar' from resultset 'Template'.
    $dbic_provider->resultset(
        $schema->resultset('Template')->search({site=>'bar'})
    );
    $template->process('other_template');

# DESCRIPTION

Template::Provider::PerContextDBIC allows a [Template](https://metacpan.org/pod/Template) object to fetch
its data using [DBIx::Class](https://metacpan.org/pod/DBIx::Class) instead of, or in addition to, the default
filesystem-based [Template::Provider](https://metacpan.org/pod/Template::Provider). The PerContextDBIC provider also
allows changing the `resultset` between calls to $template->process.

This module was inspired by both [Template::Provider::DBIC](https://metacpan.org/pod/Template::Provider::DBIC) and
[Template::Provider::PrefixDBIC](https://metacpan.org/pod/Template::Provider::PrefixDBIC). It uses ideas from both of the other
excellent modules.

# ATTRIBUTES

## COLUMN\_NAME

The table column that contains the template name. This will default to
'tmpl\_name'.

## COLUMN\_CONTENT

The table column that contains the template data itself. This will
default to 'content'.

## COLUMN\_MODIFIED

The table column that contains the date that the template was last
modified. This will default to 'modified'.

## RESULTSET

The resultset to be used to `find` templates. It can be left blank as
long as `$provider-`resultset(...)> is called prior to template
processing.

## RESTRICTBY\_NAME

The unique value identifying the `$resultset` to use for creating cache
directories and lookups. Can be left blank if `$resultset` has a
`restricting_object` method (eg. using
[DBIx::Class::Schema::RestrictWithObject](https://metacpan.org/pod/DBIx::Class::Schema::RestrictWithObject)).

## RESULTSET\_METHOD

The sub reference to be called during `fetch` which will return a two
item list with `$resultset` and `$restrictby_name`.

## TOLERANT\_QUERY

If set to a true value, then a query with more than one row will cause
the provider to return `STATUS_DECLINED` rather than `STATUS_ERROR`.

# METHODS

## ->\_init( \\%options )

Check that valid Template::Provider::PerContextDBIC-specific arguments
have been supplied and store the appropriate values. See above for the
available options.

## ->lookup\_name( $name )

This method returns the name of the template that will be looked up in
the database. The `TEMPLATE_EXTENSION` will be removed as well as the
leading table name and restricting object.

## ->cache\_name( $name )

This method returns the name of the cache entry. It will have the
leading table name and restricting object as part of the name.

## ->fetch( $name )

This method is called automatically during [Template](https://metacpan.org/pod/Template)'s
`->process()` and returns a compiled template for the given
`$name`, using the cache where possible.

## ->\_load( $name )

Load the template from the database and return a hash containing its
name, content, the time it was last modified, and the time it was loaded
(now).

## ->\_modified( $name, $time )

When called with a single argument, returns the modification time of the
given template. When called with a second argument it returns true if
$name has been modified since $time.

## ->resultset($rs, \[$restrict\_by\])

Pass a resultset to use for subsequent template processing. Optionally
pass a string to use for making unique cache directory.

If `$restrict_by` is not set, and the `$rs` has a
`restricting_object` method (eg. using
[DBIx::Class::Schema::RestrictWithObject](https://metacpan.org/pod/DBIx::Class::Schema::RestrictWithObject)), then `$restrict_by` will
be set to a string containing the
`$restricting_object-`result\_source->name> and
`$restricting_object-`id>.

If both `$restrict_by` is set, and the `$rs` has a
`restricting_object` method, then both values are used to create a
unique cache directory.

## ->get\_template($lookup\_name, @columns)

Pass a template name to retrieve from the database, as well as list of
columuns to be included. The column names are specified as the keys
`COLUMN_CONTENT` and `COLUMN_MODIFIED`. A `$row` will be returned if
lookup\_name matches a record.

The database query is done using `search` rather than `find` to avoid
warnings when query returns more than one record. The author feels that
`find` is the correct method, but puts too much burden to ensure unique
queries.

The default behaviour is to return a `STATUS_ERROR` if more than one
row is returned. You can change that behaviour with the
`TOLERANT_QUERY` option.

# USE WITH OTHER PROVIDERS

By default Template::Provider::PerContextDBIC will raise an exception
when it cannot find the named template. When TOLERANT is set to true it
will defer processing to the next provider specified in LOAD\_TEMPLATES
where available. For example:

    my $template = Template->new({
        LOAD_TEMPLATES => [
            Template::Provider::PerContextDBIC->new({
                RESULTSET => $resultset,
                TOLERANT  => 1,
            }),
            Template::Provider->new({
                INCLUDE_PATH => $path_to_templates,
            }),
        ],
    });

# CACHING

When caching is enabled, by setting COMPILE\_DIR and/or COMPILE\_EXT,
Template::Provider::PerContextDBIC will create a directory consisting of
the database DSN and table name, and restrict\_by name. This should
prevent conflicts with other databases and providers.

In addition, if the result set has been restricted using
[DBIx::Class::Schema::RestrictWithObject](https://metacpan.org/pod/DBIx::Class::Schema::RestrictWithObject), the cache directory will
also be prefixed with the name and id of the restricting object. This
should prevent conflicts with other resultsets for the same table. 

# SEE ALSO

- [Template::Provider](https://metacpan.org/pod/Template::Provider)
- [Template::Provider::DBIC](https://metacpan.org/pod/Template::Provider::DBIC)
- [Template::Provider::PrefixDBIC](https://metacpan.org/pod/Template::Provider::PrefixDBIC)
- [DBIx::Class::Schema](https://metacpan.org/pod/DBIx::Class::Schema)

# DIAGNOSTICS

In addition to errors raised by [Template::Provider](https://metacpan.org/pod/Template::Provider) and [DBIx::Class](https://metacpan.org/pod/DBIx::Class),
Template::Provider::DBIC may generate the following error messages:

- `does not support the SCHEMA option`

    The SCHEMA configuration option should not be provided.

- `You must provide a DBIx::Class::ResultSet before calling fetch`

    Couldn't find a valid resultset when $provider->fetch runs.

- `More then one template matching '%s' was found in the resultset`

    The template %s was found more than once when querying the resultet. 

- `Could not retrieve '%s' from the result set '%s'`

    Unless TOLERANT is set to true failure to find a template with the given name
    will raise an exception.

# DEPENDENCIES

- [Carp](https://metacpan.org/pod/Carp)
- [Date::Parse](https://metacpan.org/pod/Date::Parse)
- [File::Path](https://metacpan.org/pod/File::Path)
- [File::Spec](https://metacpan.org/pod/File::Spec)
- [Template::Provider](https://metacpan.org/pod/Template::Provider)

Additionally, use of this module requires an object of the class
 [DBIx::Class::ResultSet](https://metacpan.org/pod/DBIx::Class::ResultSet).

# AUTHOR

Charlie Garrison <garrison@zeta.org.au>

# COPYRIGHT AND LICENSE

This software is copyright (c) 2015 by Charlie Garrison.

This is free software; you can redistribute it and/or modify it under
the same terms as the Perl 5 programming language system itself.
