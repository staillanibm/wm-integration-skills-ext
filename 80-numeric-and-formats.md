# Decimal arithmetic and number formats

Applies whenever a service computes or formats amounts, prices, rates or quantities.

## Never compute money in floating point

`pub.math:addFloats`, `multiplyFloats`, `divideFloats`, `subtractFloats` work on `double`:
`0.1 + 0.2 = 0.30000000000000004`, `1.1 x 3 = 3.3000000000000003`, and a product that is exactly a
half (`3 x 0.335 = 1.005`) can land just below it and round the wrong way.

Exact products:

1. `pub.math:toNumber` with `convertAs = "java.math.BigDecimal"` on each operand (input `num`, output
   `num` as `Object`: copy it to a differently named `Object` variable at once).
2. `pub.math:multiplyObjects` / `addObjects` (`num1`, `num2` as `Object`, output `value`).
3. `pub.string:objectToString` (`object` -> `string`), then `pub.math:roundNumber`.

Wrap this chain in **one small service** (`multiplyDecimals(factor1, factor2, factor3) -> product`) and
reuse it everywhere, so the rounding rule lives in one place. A percentage is a product with
`0.01` as third factor.

Sums of values that are already rounded to the target scale may stay in `addFloats`: the error is far
below half a unit of the last place. Format the sum afterwards.

## Rounding

`pub.math:roundNumber` takes `num`, `numberOfDigits` (not `numDecimalPlaces`) and optionally
`roundingMode` (`roundHalfUp`, `roundHalfEven`, ...). It rounds from the decimal text, so `1.005` with
`roundHalfUp` gives `1.01`. Negative values round half away from zero (`-2.345` gives `-2.35`).
Round **once**, at the level the business rule names (e.g. VAT once per rate on the cumulated base,
not once per line).

## Formatting

- `pub.string:numericFormat` with pattern `0.00` pads to two decimals but **follows the server
  locale**: `9,90` with a comma, even when `language` is set. Replace the comma afterwards with
  `pub.string:replace` (`inString`, `searchString`, `replaceString`): it is a no-op when the locale
  already uses a dot.
- Format with a fixed pattern at the very end of the computation; keep intermediate values unformatted.
- An integer such as `5` must become `5.00`, `255.0` (a double rendered by `addFloats`) must become
  `255.00`: test both.

## Tests worth writing for a numeric service

An exact half (`0.005`, `1.005`), a value a `double` gets wrong (`3 x 0.335`), an integer, a value
with a trailing `.0`, a negative, zero, more decimals than the scale, and a non-numeric input (the
message comes from the platform: read it by invoking once, do not guess it).
