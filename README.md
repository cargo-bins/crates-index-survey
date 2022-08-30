# Tools to survey the crates.io registry

```
/!\ While these tools hit the CDN directly, you should still refrain from using
them except than "occasionally", so as to not disrupt crates.io operations. /!\
```

These tools were written specifically for investigating Cargo Binstall and make
informed decisions about planned features. They are not maintained and may not
suit your usecase. Caveat emptor.

## Building

These crates require building with `RUSTFLAGS="--cfg tokio_unstable"`, and it's
highly recommended to build in release mode.

## `fetch-crates`

Updates the local cargo index, traverses it to get a list of all crates at
their latest non-prerelease versions, and then downloads-and-unpacks the crates
in the `./crates` folder, one `cratename-version` folder per.

It's highly recommended to have ample free space, a fast internet connection,
and filesystem compression enabled. At writing the tool downloads about 11
gigabytes of data, and writes roughly 45 gigabytes to the filesystem. With
Btrfs ZSTD compression, that works out to about 15 gigabytes of disk usage.

On decent fibre broadband and an SSD it should take less than half an hour.

## `load-crates`

Uses the `DATABASE_URL` to open a connection to a postgres database, which must
have this table created:

```sql
CREATE TABLE crates (
    name varchar(200) not null,
    version varchar(100) not null,
    manifest jsonb not null
);
```

It then traverses the `./crates` folder as prepared by `fetch-crates`, resolves
the full Cargo manifest from the `Cargo.toml` and the `src` folder for each
crate, converts that structure to JSON, and finally loads it into the database.

It is then possible to use standard PostgreSQL queries to interrogate the registry.

## Examples

### Count all crates

```sql
select count(*) from crates;
```

### Count of binary crates

```sql
select count(*) from crates where manifest->'bin' is not null;
```

### List crates with a pijul repository and their authors

```sql
select name, version, manifest->'package'->'authors'
from crates
where (manifest->'package'->'repository')::text like '%pijul%';
```

### Summarise crate counts by the hostname of their repository field

```sql
select * from (
	select
		substring((manifest->'package'->'repository')::text from '://[^/]+') hostname,
		count(*) count
	from crates
	where true
		and manifest->'package'->'metadata'->'binstall' is null
		and manifest->'package'->'repository' != 'null'::jsonb
	group by substring((manifest->'package'->'repository')::text from '://[^/]+')
) n
order by count desc;
```

### Show non-Binstall binary crates with a repository field which uses plain HTTP

```sql
select name, manifest->'package'->'repository'
from crates
where true
	and manifest->'package'->'metadata'->'binstall' is null
	and manifest->'package'->'repository' != 'null'::jsonb
	and substring(
		(manifest->'package'->'repository')::text from '^[^:]+:'
	) = '"http:'
	and manifest->'bin' is not null;
```

