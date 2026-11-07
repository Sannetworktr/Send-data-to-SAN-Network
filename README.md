# Send data to SAN Network

Send ADS-B aircraft positions, AIS vessel positions, or VHF MP3 segments over HTTPS.

## 1. Create an ingest key

1. Open [Member Panel → Send data](https://sannetwork.net/member?lang=en#send).
2. Choose ADS-B, AIS, or VHF. Read and accept the data-sharing terms.
3. Create your key and store it securely. Copy the full value when it is shown at creation. Keep it out of repositories, logs, and screenshots.
4. Add an IP allowlist if needed. Key rotation immediately invalidates the old key. Update your sender configuration with the replacement.

## 2. Send your first observation

The examples use Unix seconds. Set the environment variables to actual observations from your receiver and provide the ingest key securely.

### ADS-B

```bash
jq -n --argjson now "$(date +%s)" --arg hex "$AIRCRAFT_HEX" \
  --argjson lat "$AIRCRAFT_LAT" --argjson lon "$AIRCRAFT_LON" \
  --argjson seen "$AIRCRAFT_AGE_SECONDS" \
  '{now:$now,aircraft:[{hex:$hex,lat:$lat,lon:$lon,seen:$seen}]}' |
curl -sS --fail-with-body 'https://sannetwork.net/api/contribute/adsb' \
  -H "X-Ingest-Key: $SAN_INGEST_KEY" -H 'Content-Type: application/json' \
  --data-binary @-
```

`hex` is a six-character hexadecimal ICAO address. `seen` is the age of the observation in seconds (0–60).

### AIS

```bash
jq -n --argjson now "$(date +%s)" --arg mmsi "$VESSEL_MMSI" \
  --argjson lat "$VESSEL_LAT" --argjson lon "$VESSEL_LON" \
  '{now:$now,vessels:[{mmsi:$mmsi,lat:$lat,lon:$lon}]}' |
curl -sS --fail-with-body 'https://sannetwork.net/api/contribute/ais' \
  -H "X-Ingest-Key: $SAN_INGEST_KEY" -H 'Content-Type: application/json' \
  --data-binary @-
```

`mmsi` is a nine-digit identifier. For ADS-B and AIS, provide latitude and longitude in decimal degrees and `now` in Unix seconds within 120 seconds of the server clock. The JSON payload can contain up to 1,000 records and must be at most 250,000 bytes.

### VHF

```bash
curl -sS --fail-with-body 'https://sannetwork.net/api/contribute/vhf' \
  -H "X-Ingest-Key: $SAN_INGEST_KEY" -H 'Content-Type: audio/mpeg' \
  --data-binary @segment.mp3
```

Send MP3 segments over HTTPS. Each segment must be 1,000–512,000 bytes.

## 3. Link and verify your station

A successful response includes `stationId`. Open [Connect station](https://sannetwork.net/member?lang=en#claim), select the station, enter the receiver details, and save. In [My stations](https://sannetwork.net/member?lang=en#stations), check the last received time and your ADS-B or AIS positions. For VHF, check the station status and available audio segment.

Wait at least 10 seconds between submissions with the same key. If you receive `429`, wait for the indicated interval before retrying.

| Response | What to check |
| --- | --- |
| `401` | The ingest key and its service. |
| `403` | Key status and the IP allowlist. |
| `413` | Payload size. |
| `415` | Audio format and content type. |
| `422` | Timestamp, coordinates, and required fields. |
