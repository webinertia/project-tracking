# Webware-Core Changes: Session Handoff (2026-09-03)

> Written from the webware-acl session so the **webware-core** session can pick
> up the one shared change required to unblock the laminas-inputfilter 3.0
> `validate()` migration. Read this first; it is the authoritative "what core
> needs" snapshot.

## TL;DR

- `SystemMessageTrait` (`vendor` path: `src/InputFilter/SystemMessageTrait.php`)
  must change signature so it formats an `ErrorMessages` object **passed in**,
  instead of reading `$this->getMessages()` off the input filter.
- Reason: laminas-inputfilter 3.0's stateless `validate()` returns a result
  object; the filter itself no longer holds validation messages. The old
  stateful `getMessages()` is `@deprecated Since 3.0` and removed in 4.0.
- This is shared across **webware-acl (first)**, **webware-usermanager**, and
  **ims-\*** components - hence it belongs in core, not duplicated per package.

---

## 1. Why

laminas-inputfilter 3.0 replaced the stateful API
(`setData()` → `isValid()` → `getValues()` / `getMessages()`) with a stateless
one:

```php
public function validate(iterable $data, array $context = []): InputFilterValidationResult;
```

- The result carries `valid()`, `value()` (filtered), `rawValue()` (unfiltered),
  and `getMessages(): ErrorMessages`.
- The filter object no longer holds messages between calls, so the trait's
  current `$this->getMessages()` has nothing to read after `validate()`.

The trait must therefore accept the `ErrorMessages` from the result as an
argument.

---

## 2. Change required (webware-core)

File: `src/InputFilter/SystemMessageTrait.php`

### Before

```php
<?php

declare(strict_types=1);

namespace Webware\Core\InputFilter;

use function implode;
use function is_array;
use function is_string;
use function json_encode;

/**
 * @api
 * @mixin \Laminas\InputFilter\InputFilterInterface
 */
trait SystemMessageTrait
{
    public function getSystemMessage(bool $asJson = false): string
    {
        if ($asJson) {
            $encoded = json_encode($this->getMessages()->jsonSerialize());

            return false === $encoded ? '[]' : $encoded;
        }

        /** @var array<array-key, string|array<array-key, string|array<array-key, string>>> $messages */
        $messages = $this->getMessages()->toArray();

        return implode('<br />', $this->flattenMessages($messages));
    }

    // flattenMessages() + isLeafMessageSet() unchanged
}
```

### After

```php
<?php

declare(strict_types=1);

namespace Webware\Core\InputFilter;

use Laminas\InputFilter\ErrorMessages;

use function implode;
use function is_array;
use function is_string;
use function json_encode;

/**
 * @api
 */
trait SystemMessageTrait
{
    public function getSystemMessage(ErrorMessages $messages, bool $asJson = false): string
    {
        if ($asJson) {
            $encoded = json_encode($messages->jsonSerialize());

            return false === $encoded ? '[]' : $encoded;
        }

        /** @var array<array-key, string|array<array-key, string|array<array-key, string>>> $flat */
        $flat = $messages->toArray();

        return implode('<br />', $this->flattenMessages($flat));
    }

    // flattenMessages() + isLeafMessageSet() unchanged
}
```

Notes:

- Add `use Laminas\InputFilter\ErrorMessages;` import.
- Remove the now-inaccurate `@mixin \Laminas\InputFilter\InputFilterInterface`
  annotation (the trait no longer calls `$this->getMessages()`).
- `ErrorMessages` is `final readonly` with public `toArray()` and
  `jsonSerialize()` - no interface changes needed.
- `flattenMessages()` / `isLeafMessageSet()` are unchanged.

---

## 3. Consumer contract (what call sites become)

```php
$result = $filter->validate($data);

if (! $result->valid()) {
    $messenger?->warning($filter->getSystemMessage($result->getMessages()));

    return $handler->handle($request);
}

$values = $result->value();
```

This is the exact shape webware-acl (and later usermanager / ims-\*) will use.

---

## 4. Ordering

1. webware-core: apply the trait change (this handoff).
2. webware-acl: `composer update webware/webware-core`, then finish issue #23
   (the middleware + filter migration already references the new signature).
3. webware-usermanager / ims-\*: same trait reuse when they migrate.
