# Interrupts in RPi::Pin

`RPi::Pin` represents a single GPIO pin. You **arm** an interrupt on the pin, but
the event is **dispatched through the Pi object** (`RPi::Pin` is normally used
through [RPi::WiringPi](https://metacpan.org/pod/RPi::WiringPi) — you get pins
with `my $pin = $pi->pin($num)`).

This pin object gives you two interrupt methods:

- `set_interrupt($edge, $callback, $debounce_us, \%opts)` — arm an interrupt.
- `background_interrupt($edge, $callback, $debounce_us, \%opts)` — handle it in a
  forked background process.

Everything that *drives* dispatch (the loop, auto-dispatch, teardown) lives on
the Pi object `$pi`, because the event queue is process-wide. See
[RPi::WiringPi](https://metacpan.org/pod/RPi::WiringPi)'s interrupt methods for
those.

## The dispatch gotcha (read this first)

As of `WiringPi::API` 3.18 a callback does **not** fire on its own. The wiringPi
ISR thread only queues the edge; your callback runs in your own interpreter
**when you service dispatch**. Two consequences:

1. The callback **must be a code reference** (`\&handler` or `sub {...}`). The
   old string sub-name form (`'main::handler'`) is **no longer accepted** — this
   is a breaking change.
2. After arming you must drive dispatch, via the Pi object:

```perl
$pin->set_interrupt(EDGE_RISING, \&handler);

$pi->wait_interrupts(1000) while 1;   # or $pi->run_interrupt_loop(1000)
# ... or $pi->dispatch_interrupts inside your own event loop ...
```

The easiest "fire and forget" path is to let the library service it for you —
either enable process-wide auto-dispatch on the Pi object, or opt in while
arming:

```perl
$pi->auto_dispatch_interrupts(1);                              # global switch
# or, as part of arming this pin:
$pin->set_interrupt(EDGE_RISING, \&handler, { auto_dispatch => 1 });
```

## A full example

```perl
use RPi::WiringPi;
use RPi::Const qw(:all);

my $pi  = RPi::WiringPi->new;
my $pin = $pi->pin(18);
$pin->mode(INPUT);

$pin->set_interrupt(EDGE_RISING, \&on_rise);

$pi->wait_interrupts(1000) while 1;

sub on_rise {
    my ($edge, $ts_us) = @_;     # the callback always gets these two args
    print "rising edge at $ts_us us\n";
}
```

## `set_interrupt` in detail

```perl
$pin->set_interrupt($edge, $callback, $debounce_us, \%opts);
```

- **`$edge`** — `EDGE_FALLING` (1), `EDGE_RISING` (2) or `EDGE_BOTH` (3), from
  `RPi::Const qw(:all)`.
- **`$callback`** — a **code reference**. It receives `($edge, $timestamp_us)`.
- **`$debounce_us`** *(optional)* — debounce window in microseconds; edges within
  this window of the previous accepted edge are ignored (kernel debounce).
- **`\%opts`** *(optional)* — forwarded to `WiringPi::API`. The `auto_dispatch`
  option turns on auto-dispatch as part of arming (process-wide):

```perl
$pin->set_interrupt(EDGE_RISING, \&handler, { auto_dispatch => 1 });
# choose the delivery signal (avoids clashing with other SIGIO users):
$pin->set_interrupt(EDGE_RISING, \&handler, { auto_dispatch => 'USR1' });
```

Re-arming the same pin swaps the handler cleanly (the old listener is stopped
first). Tear down from the Pi object: `$pi->stop_interrupts` (or the Pi's
`cleanup`, which also releases interrupts).

## `background_interrupt` (run in the background)

Handle the interrupt in a forked child that fires even while your main program
is busy. The handler runs in the child, so it **cannot** touch your main
program's variables — use it for independent work.

```perl
my $h = $pin->background_interrupt(EDGE_RISING, sub {
    my ($edge, $ts_us) = @_;
    # independent handler, in the background
});

# ... main carries on ...

$h->stop;          # stop + reap (idempotent)
$h->pid;           # the child PID
$h->running;       # true while alive
```

### Reporting values back (the `results` channel)

Pass `{ results => 1 }` and **return** a value from the handler; the parent
drains it:

```perl
my $h = $pin->background_interrupt(EDGE_RISING, sub {
    my ($edge, $ts_us) = @_;
    return "$edge\@$ts_us";
}, { results => 1 });

while (defined(my $msg = $h->read)) {   # non-blocking
    print "from handler: $msg\n";
}
# $h->fh is the read filehandle (for select / IO::Select)
```

## Deprecated: `interrupt_set`

`$pin->interrupt_set($edge, $callback)` is **deprecated** — use `set_interrupt`
instead. It remains only for backward compatibility.

## Where the rest lives

Anything that isn't pin-specific is on the **Pi object** `$pi`, because the
interrupt queue is shared across the whole process:

| Need | Use (on `$pi`) |
|---|---|
| Run/service dispatch | `wait_interrupts`, `run_interrupt_loop` / `stop_interrupt_loop`, `dispatch_interrupts` |
| Fire callbacks with no loop | `auto_dispatch_interrupts($bool [, $signal])` |
| Many pins in one background child | `background_interrupts([...], ...)` |
| Which pin fired last | `last_interrupt` |
| Size the event queue | `interrupt_buffer([$bytes])` |
| Release every interrupt | `stop_interrupts` (and `cleanup`) |

See [RPi::WiringPi](https://metacpan.org/pod/RPi::WiringPi) for those, and
[WiringPi::API](https://metacpan.org/pod/WiringPi::API) for the underlying
procedural API.
