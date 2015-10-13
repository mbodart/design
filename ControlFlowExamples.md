#### If with no else:

```
(if (cond-expr)
  (then-body)
)
```

#### If with else:

```
(if (cond-expr) $exit $else
  (then-body)
  (br $exit)
  $else
  (else-body)
)
```

#### If-else chain:

```
(if (cond-expr0) $exit0 $else0
  (then-body0)
  (br $exit0)
  $else0
  (if (cond-expr1) $exit1 $else1
    (then-body1)
    (br $exit0)
    $else1
    (else-body)
  )
)
```

### simple br_if example:

```
(br_if $label (cond-expr))
```

#### Switch with 3 internal cases and an internal default.

```
(switch (index-expr) $exitlabel
  ($internal0 $internal1 $internal2) $default
  (case $internal0
    ((internal-body0) (br $exitlabel)))
  (case $internal1
    (internal-body1)) ;; possible fallthru
  (case $internal2
    ((internal-body2) (br $exitlabel)))
  (case $default
    (default-body))
)
```

#### Switch with 3 internal cases, 3 external cases, and an external default.

```
(block $external0
  (block $external1
    (block $external2
      (block $default
        (switch (index-expr) $exitlabel
          ($external0 $external1 $external1 $internal0 $internal1 $internal2) $default
          (case $internal0
            ((internal-body0) (br $exitlabel)))
          (case $internal1
            (internal-body1)) ;; possible fallthru
          (case $internal2
            ((internal-body2) (br $exitlabel)))
        )
        (stuff)
      )
      (more stuff)
    )
    (other stuff)
  )
  (things)
)
```

#### Infinite loop

```
(loop $exitlabel $top
  (body)
  (br $top)
)
```

#### Do-while loop

```
(loop $exitlabel $top
  (body)
  (br_if label $top)
)
```

#### While loop:

```
(if (not (cond))
  (loop $exitlabel $top
    (body)
    (br_if $top (cond))
  )
)
```
