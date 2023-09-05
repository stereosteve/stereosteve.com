---
title: "pl pg sql"
draft: true
---

# oh hey

```sql
CREATE OR REPLACE FUNCTION say_hello(name text)
RETURNS text AS $$
BEGIN

  raise notice 'saying hello to %', name;
  return 'Hello ' || name;

END; $$ LANGUAGE plpgsql;

SELECT say_hello('dave');
```
