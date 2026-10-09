# Failure Path Register: Assignment 2 - Logistics & Shipment Exception Management

This document details the formal Failure Path Register for Assignment 2, modeling all exception paths across pickup, line-haul sorting hubs, customs clearance, and last-mile delivery.

---

## 📊 Summary Matrix (E1 – E10)

| ID | Failure Point | Cause / Trigger | Solution & BPMN Construct | Recovery Strategy | Lane / Owner |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **E1** | Pickup Failure | Shipper absent, premises locked, or parcel not packaged/ready | Error Boundary Event (`BoundaryError_PickupFailed`) on Pickup User Task + Attempt Gateway | Dispatcher reschedules pickup for next available slot (max 2 attempts). If exceeded, cancels booking, charges cancellation penalty, and frees driver capacity. | Pickup Agent / Control Tower |
| **E2** | Address Validation / Delivery Ambiguity | Incomplete address, missing apartment number, or invalid postal code | Business Rule Task at booking; Doorstep User Task + Exclusive Gateway (`Gateway_DoorstepAddressSurcharge`) at delivery | At booking: system prevents dispatch, prompting shipper to correct. At doorstep: agent contacts recipient via phone/SMS; an address reroute surcharge is calculated at an exclusive gateway before re-dispatching. | Shipper / Delivery Agent |
| **E3** | Damaged Parcel | Physical container crushed, punctured, or wet at hub scan or doorstep handover | Inspection Task + Exclusive Gateway on Insurance (`Gateway_InsuranceCheck`) routing to Claims Sub-Process | Warehouse supervisor documents damage photos. If insured, launches Insurance Claims Sub-Process for customer reimbursement or replacement reshipment. If uninsured, logs liability waiver and returns to sender. | Hub Warehouse / Claims Department |
| **E4** | Lost Parcel | No barcode scan detected across any transit hub for 48 consecutive hours | Intermediate Timer Catch Event (`PT48H`) triggering 5-Day Tracing Sub-Process (`SubProcess_TracingLostParcel`) | Automated alert triggers depot CCTV investigation and physical bay audit. If parcel remains missing after 5 days, declares lost, disburses compensation, notifies customer, and ends `"Lost"`. | Hub Warehouse / Claims Department |
| **E5** | Transit Delays | Extreme weather, highway blockade, truck breakdown, or hub mechanical overload | Event-Based Gateway (`Gateway_TransitDelayOrTimer`) awaiting Delay Message or Scheduled Timer | Upon delay signal, system recalculates dynamic ETA, triggers automated push notification to customer, and assigns alternate fleet vehicle or multimodal reroute. | Control Tower / Dispatcher |
| **E6** | Customs / Cross-Border Hold | Missing commercial invoice, incorrect HS tariff codes, or export inspection hold | Send Task + User Task (`Task_UploadCustomsDocs`) with 5-Day Timer Boundary Event | Formal alert dispatched to shipper requesting missing documents. If provided within 5 days, customs cleared and transit resumes; if timer expires, parcel returned to origin country. | Hub Warehouse / Shipper |
| **E7** | Recipient Unavailable | Recipient not home, security gate refusal, or unreachable by phone | Exclusive Gateway (`Gateway_DeliveryAttempts`) with delivery attempt counter | Agent leaves digital/physical notice; delivery re-attempted next business day (up to 3 attempts). After 3 failed attempts, parcel held at depot for 7 days before returning to sender (RTS). | Delivery Agent / Hub Warehouse |
| **E8** | Recipient Refusal / COD Failure | Recipient rejects item, dispute over contents, or cash-on-delivery payment fails | Exclusive Gateway (`Gateway_DeliveryOutcome`) routing to RTS End Event | Agent captures formal refusal code/signature. Driver returns parcel to delivery hub, return logistics initiated, and customer billing/freight charges adjusted. | Delivery Agent / Control Tower |
| **E9** | Tracking / Scanner Malfunction | Handheld scanner battery dead, Wi-Fi blind spot, or barcode label smudged/torn | Error Boundary Event (`BoundaryError_ScannerFailure`) on Inbound Hub Scan Task | System catches scanner offline exception; hub supervisor manually keys tracking number/waybill, followed by an automated database reconciliation task. | Hub Warehouse |
| **E10** | Shipper Cancellation in Transit | Shipper recalls shipment after dispatch due to order cancellation or fraud prevention | Interrupting Message Boundary Event (`BoundaryMessage_ShipperCancellation`) on Transit Sub-Process with Compensation | Message event halts physical movement at next hub; triggers compensation event (`Event_TriggerCompensationCancellation`) calculating refund minus transit freight fee, and reroutes parcel back to shipper. | Shipper / Hub Warehouse |

---

## 🔍 Detailed Exception Handling Architecture

### E1: Pickup Failures & Rescheduling Limits
- **Trigger**: Driver arrives at pickup premises; shipper is absent or parcel is unready.
- **BPMN Construct**: Error Boundary Event (`BoundaryError_PickupFailed`) attached to `Task_CollectAndScanParcel`, routing to `Task_HandlePickupException` and Exclusive Gateway `Gateway_PickupAttempts`.
- **Handling**:
  - `pickupAttempts <= 2`: System reschedules pickup window and reassigns driver route.
  - `pickupAttempts > 2`: Process terminates at `EndEvent_BookingCancelledFee`, assessing a missed pickup charge to the customer's account.
- **Rationale**: Bounding attempts prevents driver route disruption and operational fuel waste.

### E2: Address Validation (Intake vs. Last-Mile Doorstep)
- **Intake**: Automated postal service API verifies address format during booking. If invalid (`isAddressValid == false`), user task prompts customer correction before driver dispatch.
- **Doorstep (Last-Mile)**: Delivery agent arrives but discovers street address is ambiguous. Agent uses handheld app to contact recipient (`Task_ContactRecipientDoorstep`).
- **Surcharge Evaluation**: If recipient requests redirection to a new delivery address across town, Exclusive Gateway (`Gateway_DoorstepAddressSurcharge`) checks whether reroute fee is accepted (`surchargeAccepted == true`). If accepted, delivery attempt counter is incremented and rescheduled; if rejected, package is returned to sender.

### E3: Damaged Parcel & Claims Sub-Process
- **Trigger**: Barcode scan or visual inspection identifies crushed carton or leaking contents.
- **BPMN Construct**: `Task_InspectDamagedParcel` followed by Exclusive Gateway `Gateway_InsuranceCheck`.
- **Handling**:
  - `isInsured == true`: Enters `SubProcess_ClaimsSettlement`. A claims officer verifies declared value, issues customer payout or triggers replacement reshipment, and writes off damaged goods.
  - `isInsured == false`: Shipper is notified with photographic evidence; parcel returned or scrapped per carriage agreement.

### E4: Missing Scan / Lost Parcel Tracing
- **Trigger**: Normal transit between Hub A and Hub B exceeds 48 hours without intermediate scan.
- **BPMN Construct**: Event-Based Gateway catches 48-hour missing scan timer, triggering `SubProcess_TracingLostParcel`.
- **Handling**: Hub warehouse staff review bay surveillance and line-haul GPS tracking. An intermediate timer waits 5 days for investigation. If parcel cannot be located, `Task_DeclareLostAndCompensate` processes statutory courier compensation and marks shipment terminal state as `"Shipment Closed as Lost"`.

### E5: In-Transit Delays & ETA Dynamic Recalculation
- **Trigger**: Severe winter storm closes transit corridors or line-haul truck suffers engine breakdown.
- **BPMN Construct**: Event-Based Gateway (`Gateway_TransitDelayOrTimer`) listens concurrently for normal arrival timer vs. external delay message (`Message_TransitDelay`).
- **Handling**: Upon arrival of delay message, `Task_NotifyCustomerDelayETA` updates real-time tracking interface, and `Task_RerouteTransitVehicle` assigns alternative transit routing.

### E6: Customs Clearance & Document Hold
- **Trigger**: Cross-border shipment held by customs authorities due to missing commercial invoice.
- **BPMN Construct**: Exclusive Gateway checks `isInternational`. If customs hold occurs, `Task_RequestCustomsDocuments` notifies shipper. A 5-day interrupting timer boundary event monitors document provision.
- **Handling**: If documents provided, goods clear customs; if 5 days elapse, international carrier automatically returns parcel to origin country.

### E7: Recipient Unavailable & Depot Holding Period
- **Trigger**: Recipient not present at delivery address during delivery attempt.
- **BPMN Construct**: Exclusive Gateway (`Gateway_DeliveryAttempts`) tracks `deliveryAttempts`.
- **Handling**: Driver logs doorstep attempt and customer receives SMS/email. Loop allows up to 3 delivery attempts. After the 3rd attempt, parcel moves to `Task_DepotHold7Days` allowing customer self-pickup. If unclaimed after 7 days, package automatically routes to Return to Sender.

### E8: Customer Refusal & Cash-On-Delivery Failure
- **Trigger**: Recipient refuses delivery (wrong item ordered) or fails to provide cash/card for COD payment.
- **BPMN Construct**: Exclusive Gateway `Gateway_DeliveryOutcome` routes directly to `EndEvent_ShipmentReturnedToSender`.
- **Handling**: Driver secures item, logs refusal code in handheld scanner, and carrier bills shipper for return freight.

### E9: Handheld Scanner & Telemetry Failures
- **Trigger**: Scanner battery dies, barcode is torn, or cloud synchronization fails during hub sorting.
- **BPMN Construct**: Error Boundary Event (`BoundaryError_ScannerFailure`) catches `ERR_SCANNER_OFFLINE`.
- **Handling**: Hub supervisor manually inputs waybill number into web portal. Once connectivity is restored, automated reconciliation task reconciles physical inventory with digital logistics manifest.

### E10: Shipper In-Transit Cancellation with Transaction Compensation
- **Trigger**: Merchant detects fraudulent order and issues instant recall request via API.
- **BPMN Construct**: Interrupting Message Boundary Event (`BoundaryMessage_ShipperCancellation`) on `SubProcess_HubLineHaulTransit` triggering `Event_TriggerCompensationCancellation`.
- **Handling**: Compensation task `Task_CompensateHaltAndReturnParcel` commands destination hub to flag parcel for immediate reversal, calculates partial refund minus return freight fees, and terminates at Return to Sender.
