[![Maintained by Avalux](https://img.shields.io/badge/Maintained%20by-Avalux.io-3b82f6?style=flat-square)](https://avalux.io)

# Recipe: Auto-Text Freight Receiver When Truck Is One Hour Out

> The classic SMB freight brokerage problem: receivers complain that they never know when a truck is actually arriving, dispatchers spend half their day on the phone, and trucks routinely sit at docks waiting because the receiver wasn't ready. The fix is a geofence + SMS.

## The problem

A broker books a load with a carrier. The pickup is at 10am Tuesday at facility A, the drop is at 4pm Wednesday at facility B. The receiver at facility B has 30 doors and wants to know when this specific truck is one hour out so they can have a dock ready.

Without automation: dispatcher calls the driver every two hours, driver answers half the time, dispatcher relays ETAs to the receiver. Each load eats 20-40 minutes of dispatcher time. At 30 loads/day, that's a full-time job.

## Who has this problem

- Small freight brokerages running 10-200 loads/day
- 3PLs with their own fleet on telematics (Geotab, Samsara, Motive)
- Owner-operators carrying for shippers who require visibility
- Any operator stuck between "too small for Project44/FourKites" and "too big for spreadsheets"

## When this fix makes sense

- Your carriers run telematics with an API (Geotab MyGeotab, Samsara, Motive)
- You have a Twilio account or equivalent SMS API
- Drop addresses are stable enough to draw a circular or polygonal geofence around

## When it doesn't make sense

- Your carriers don't use telematics (no GPS = no geofence trigger possible)
- All your loads route through a TMS that already includes visibility (use it)
- Volume is too low to justify the setup (<5 loads/day)

## The free DIY path

Four moving parts:

1. **Poller** — Every 60-120 seconds, pull current GPS coordinates for in-transit trucks from your telematics API
2. **Geofence check** — For each truck, compute the distance to its drop facility geofence
3. **Trigger** — When the truck crosses the "1 hour out" threshold (based on speed × distance), fire the SMS
4. **State** — Make sure each truck/load combination only fires once (idempotency)

Minimum-viable Python implementation (~80 lines), using Geotab as the example. Full version with multi-carrier support, polygon geofences, and EDI 214 emission lives in our [freight-eta-toolkit](https://github.com/elikem2021/freight-eta-toolkit) repo.

```python
# freight_eta_text.py
import math, json, time, requests, os
from datetime import datetime
from twilio.rest import Client

GEOTAB_USER = os.environ["GEOTAB_USER"]
GEOTAB_PASS = os.environ["GEOTAB_PASS"]
GEOTAB_DB = os.environ["GEOTAB_DB"]
TWILIO_SID = os.environ["TWILIO_SID"]
TWILIO_TOKEN = os.environ["TWILIO_TOKEN"]
TWILIO_FROM = os.environ["TWILIO_FROM"]

ALERTED_FILE = "/var/lib/freight-eta/alerted.json"
LOADS_FILE = "/var/lib/freight-eta/active_loads.json"  # your TMS exports this nightly

tw = Client(TWILIO_SID, TWILIO_TOKEN)


def haversine_km(lat1, lon1, lat2, lon2):
    R = 6371.0
    phi1, phi2 = math.radians(lat1), math.radians(lat2)
    dphi = math.radians(lat2 - lat1)
    dlam = math.radians(lon2 - lon1)
    a = math.sin(dphi/2)**2 + math.cos(phi1) * math.cos(phi2) * math.sin(dlam/2)**2
    return 2 * R * math.asin(math.sqrt(a))


def get_geotab_vehicle_positions():
    """Returns list of {vehicle_id, lat, lon, speed_kmh}."""
    auth = requests.post("https://my.geotab.com/apiv1", json={
        "method": "Authenticate",
        "params": {"userName": GEOTAB_USER, "password": GEOTAB_PASS, "database": GEOTAB_DB},
    }, timeout=10).json()["result"]
    server, cred = auth["path"], auth["credentials"]
    positions = requests.post(f"https://{server}/apiv1", json={
        "method": "Get",
        "params": {"typeName": "DeviceStatusInfo", "credentials": cred},
    }, timeout=15).json()["result"]
    return [
        {
            "vehicle_id": p["device"]["id"],
            "lat": p["latitude"],
            "lon": p["longitude"],
            "speed_kmh": p.get("speed", 0),
        }
        for p in positions
    ]


def main():
    alerted = json.load(open(ALERTED_FILE)) if os.path.exists(ALERTED_FILE) else {}
    loads = json.load(open(LOADS_FILE))  # [{load_id, vehicle_id, drop_lat, drop_lon, receiver_phone, receiver_name}]
    positions = {p["vehicle_id"]: p for p in get_geotab_vehicle_positions()}

    for load in loads:
        if load["load_id"] in alerted:
            continue
        pos = positions.get(load["vehicle_id"])
        if not pos:
            continue
        distance_km = haversine_km(pos["lat"], pos["lon"], load["drop_lat"], load["drop_lon"])
        speed = max(pos["speed_kmh"], 40)  # floor at 40 km/h so a parked truck doesn't trigger
        eta_min = (distance_km / speed) * 60

        if 50 < eta_min < 75:  # truck is roughly one hour out
            tw.messages.create(
                to=load["receiver_phone"],
                from_=TWILIO_FROM,
                body=(
                    f"Hi {load['receiver_name']}, load {load['load_id']} is "
                    f"approximately 1 hour from your facility (currently {distance_km:.0f} km out). "
                    f"Reply STOP to opt out."
                ),
            )
            alerted[load["load_id"]] = datetime.utcnow().isoformat()
            print(f"Alerted receiver for load {load['load_id']}")

    json.dump(alerted, open(ALERTED_FILE, "w"))


if __name__ == "__main__":
    main()
```

Run as a cron job every 90 seconds:

```bash
* * * * * /usr/bin/python3 /opt/freight-eta-text/freight_eta_text.py
* * * * * sleep 30 && /usr/bin/python3 /opt/freight-eta-text/freight_eta_text.py
* * * * * sleep 60 && /usr/bin/python3 /opt/freight-eta-text/freight_eta_text.py
```

(Or once a minute is plenty — diminishing returns past that on telematics polling.)

## Common gotchas

1. **GPS jitter.** Geotab/Samsara/Motive update every 10-60 seconds. A truck at a red light might report 0 km/h — don't divide by zero. Always floor speed at a reasonable highway speed.
2. **Drop geofence shape.** Circular geofence (radius = 1 km) works fine for rural drops. For urban / DC drops with massive footprints, draw a polygon around the actual dock area.
3. **TZ handling.** Receivers often span time zones. Send timestamps in their local TZ, not UTC.
4. **Receivers don't always have stable phone numbers.** Some shipments go through brokers-of-brokers and the receiver phone is the broker's call center. Build in a fallback (email, EDI 214) if SMS fails.
5. **Carriers may not let you poll.** Some carriers gate their telematics API access behind their dispatch team. Get written permission before you start polling, or you'll get cut off mid-build.
6. **EDI 214 if shippers require it.** Larger shippers want ETA updates in EDI 214 format, not SMS. Same trigger, different output channel. See our [freight-eta-toolkit](https://github.com/elikem2021/freight-eta-toolkit) for the EDI mapping.

## Variations

- **Multi-stop loads.** Treat each stop as its own geofence with its own receiver.
- **Delay detection.** If the truck is sitting still for >15 min outside a geofence, alert the broker (not the receiver) so they can intervene.
- **2-hour-out + 30-min-out + arrived.** Run the same logic at three thresholds. Receivers love the granularity.

## Production version

[Avalux](https://avalux.io) builds the full freight visibility stack on top of this pattern — multi-carrier (Geotab + Samsara + Motive + Verizon Connect + Fleet Complete), EDI 214 emission, broker dashboards, delay detection, and a reply-handler for receiver responses. Pricing starts at $5,000. See [avalux.io/contact.html](https://avalux.io/contact.html) or email [eli@avalux.io](mailto:eli@avalux.io).

## License

MIT — part of the [solved-by-avalux](https://github.com/elikem2021/solved-by-avalux) cookbook.
