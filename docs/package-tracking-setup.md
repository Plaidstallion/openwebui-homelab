# Package Tracking Tool

A small Open-WebUI Native Function Calling Tool (`Package_Tracking_Tool.py`) for DHL, UPS, FedEx, USPS, and Hermes Germany. It's intentionally a placeholder, not a live-tracking integration — worth reading the "why" before assuming it needs finishing.

## Why it's a placeholder, not a real integration

Real tracking-API integration was scoped out after checking what each carrier actually requires:

- **DHL** has a straightforward API-key-based public API — the easiest of the five, and the one most likely to be worth doing first if this ever gets revisited.
- **UPS, FedEx, and USPS** all require a full OAuth2 client-credentials token flow — meaningfully more setup than a simple API key.
- **USPS** additionally restricts free Tracking API access to the shipper of record (as of its April 2026 Access Control changes), which rules out using it for arbitrary incoming packages you didn't ship yourself.
- **Hermes Germany** has no official public API at all — only paid third-party aggregators (TrackingMore, AfterShip) support it.

Given that spread, the tool instead does something simpler and honest: it tells the user live tracking isn't wired up, and hands back a direct link to the carrier's own tracking page instead of guessing or fabricating a status.

## The tool

```python
class Tools:
    class Valves(BaseModel):
        pass  # no configuration needed - this is a placeholder, no API credentials involved

    # generic tracking-page URLs (no query param needed) for carriers where a
    # reliable deep-link query-parameter format couldn't be confirmed, or where
    # pointing to the search page is simplest
    CARRIER_TRACKING_PAGES = {
        "dhl": "https://www.dhl.com/de-en/home/tracking.html?tracking-id={tn}",
        "ups": "https://www.ups.com/track?loc=en_US&tracknum={tn}",
        "fedex": "https://www.fedex.com/fedextrack/?trknbr={tn}",
        "usps": "https://tools.usps.com/go/TrackConfirmAction?tLabels={tn}",
        "hermes": "https://www.myhermes.de/empfangen/sendungsverfolgung/",
        "hermes germany": "https://www.myhermes.de/empfangen/sendungsverfolgung/",
    }

    def __init__(self):
        self.valves = self.Valves()

    def track_package(self, carrier: str = "", tracking_number: str = "") -> str:
        """Report on the delivery status of a package. Use this whenever the user asks
        to track a package, check a delivery status, or mentions a late/lost shipment
        (DHL, UPS, FedEx, USPS, Hermes Germany, or any other carrier) - call this
        immediately even if the carrier or tracking number hasn't been given yet, rather
        than asking the user for that information conversationally first. Pass an empty
        string for whatever isn't known yet; this tool's own response will prompt the
        user for anything missing. Always call this instead of guessing or fabricating
        a status, since live tracking data isn't wired up yet.
        :param carrier: The carrier name if known, e.g. "DHL", "UPS", "FedEx", "USPS", "Hermes". Pass an empty string if not specified or unknown.
        :param tracking_number: The tracking number if the user provided one. Pass an empty string if not given.
        :return: An honest note that live tracking isn't supported yet, plus a direct link to the carrier's own tracking page where the user can check manually.
        """
        note = (
            "Live package tracking isn't set up for this assistant yet - there's no "
            "real-time status to report. "
        )
        key = carrier.strip().lower()
        match = self.CARRIER_TRACKING_PAGES.get(key)
        if match:
            link = (
                match.format(tn=tracking_number.strip())
                if tracking_number.strip()
                else match.split("?")[0]
            )
            return (
                note
                + f"You can check it directly on {carrier.strip()}'s own tracking page: {link}"
            )
        if carrier.strip():
            return note + (
                f"I don't have a tracking page mapped for '{carrier}'. Supported carriers with "
                "known tracking pages are DHL, UPS, FedEx, USPS, and Hermes Germany."
            )
        lines = [
            note + "Depending on the carrier, try one of these tracking pages directly:"
        ]
        for name, url_template in {
            "DHL": self.CARRIER_TRACKING_PAGES["dhl"],
            "UPS": self.CARRIER_TRACKING_PAGES["ups"],
            "FedEx": self.CARRIER_TRACKING_PAGES["fedex"],
            "USPS": self.CARRIER_TRACKING_PAGES["usps"],
            "Hermes Germany": self.CARRIER_TRACKING_PAGES["hermes"],
        }.items():
            url = (
                url_template.format(tn=tracking_number.strip())
                if tracking_number.strip()
                else url_template.split("?")[0]
            )
            lines.append(f"- {name}: {url}")
        return "\n".join(lines)
```

If both `carrier` and `tracking_number` are unknown, it returns a full list of all five carriers' tracking pages rather than a dead end. If the carrier's known but not one of the five, it says so plainly instead of pretending to help.

## The bug: the tool wasn't always being called

Real-world testing surfaced an inconsistency, not a crash — the tool was attached and enabled the whole time (confirmed in the model's Tools panel), but the model wasn't reliably calling it:

- **"Can you check on my Amazon package for me? It's late."** — no tool call at all. The model just answered conversationally, asking for a tracking number and carrier in plain text, with no tool-result panel.
- **"Can you track my UPS package? It's late."** — same thing: a plain-text request for the tracking number, no tool call, even though "UPS" is one of the five carriers the tool explicitly supports.
- Only once the user supplied the tracking number in a follow-up message did the tool actually get called.

So the model was treating "do I have enough info yet?" as its own judgment call to make *before* invoking the tool — asking for missing details conversationally, then calling the tool once it already had everything. That's backwards from how the tool was designed: it already handles missing `carrier`/`tracking_number` gracefully (empty-string defaults, and a response that tells the user what's still needed).

**The fix:** the tool's docstring is what actually gets sent to the model as part of the tool schema, so it's the right place to fix model *behavior*, not just document the function. The original docstring said to use it "whenever the user asks to track a package" — true, but didn't say *when* relative to having all the details. Adding one explicit instruction fixed it:

> "...call this immediately even if the carrier or tracking number hasn't been given yet, rather than asking the user for that information conversationally first. Pass an empty string for whatever isn't known yet; this tool's own response will prompt the user for anything missing."

After this change, both previously-failing cases (the Amazon-worded request and the no-tracking-number UPS request) correctly triggered the tool call on the very first message, with the tool's own response asking for whatever was still missing.

**General takeaway:** for any Tool where you want the model to call it proactively rather than gathering information conversationally first, that instruction needs to live in the docstring, not just be assumed from the tool's parameter defaults or return behavior. The model has no visibility into the function body — only the docstring and parameter schema — so anything about *when* or *how eagerly* to call a tool has to be spelled out there explicitly.
