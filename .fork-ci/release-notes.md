StarRocks 4.1.8 plus two changes for MySQL clients such as Looker: user variables can be assigned with `:=` in the select list, which Looker generates for pivots (StarRocks/starrocks#49563), and LIKE takes an ESCAPE clause, which Looker generates for text filters. Both are FE-only, BE and CN are the stock 4.1.8 ones.

Proposed upstream in https://github.com/StarRocks/starrocks/pull/84139. This release exists to bridge the time until that is merged and released.

### Changes since 4.1.8-uservar.2

- `expr [NOT] LIKE pattern ESCAPE 'c'` is supported. The pattern must be a constant expression and the escape a single character, or an empty string for no escape character.

### Use it

Only three FE jars differ from the official 4.1.8 build: `fe-core-4.1.8.jar`, `fe-parser-4.1.8.jar` and `fe-grammar-4.1.8.jar`.

- **Docker / Kubernetes**: use `ghcr.io/marthinusbosman/starrocks/fe-ubuntu:4.1.8-uservar.3` as the FE image. It is `starrocks/fe-ubuntu:4.1.8` with the three jars replaced. Keep the official 4.1.8 BE/CN images. `ghcr.io/marthinusbosman/starrocks/allin1-ubuntu:4.1.8-uservar.3` is the same for the all-in-one image.
- **Existing 4.1.8 installation**: unpack `starrocks-fe-patched-jars-4.1.8-uservar.3.tar.gz` into `fe/lib/` of every FE, overwriting the three jars, and restart the FEs one at a time (followers first, leader last). Keep a copy of the original jars to roll back.
- **Fresh FE**: `starrocks-fe-4.1.8-uservar.3.tar.gz` is the complete `fe/` directory.

The FE reports itself as `4.1.8` with commit `b0f9467`. No metadata format changes, so switching back to the official 4.1.8 jars is a plain restart.

### What works

```sql
SELECT @rn := @rn + 1 AS rn, t.* FROM t, (SELECT @rn := 0) init ORDER BY t.c1;
```

Row numbers, rank within a group, running totals, previous-row values and the Looker pivot query from the issue return the same results as MySQL 8.0. The supported forms and the limits are described in `docs/en/sql-reference/user_defined_variables.md` on the release tag. In short: `:=` only in the select list, a variable's new value must be independent of its old value or the old value plus/minus something, and the variable keeps its pre-query value after the statement.

```sql
SELECT * FROM t WHERE c1 LIKE CONCAT('%', REPLACE(REPLACE(REPLACE('50%', '^', '^^'), '%', '^%'), '_', '^_'), '%') ESCAPE '^';
```

Looker's contains, starts with, ends with and doesn't contain filters return the same rows as MySQL 8.0 with a binary collation, also for values that hold `%`, `_`, `^` or a backslash. See `docs/en/sql-reference/sql-functions/like-predicate-functions/like.md` on the release tag.

Source: branch `uservar-4.1.8` (tag `4.1.8-uservar.3`), one commit on top of the upstream `4.1.8` tag.
