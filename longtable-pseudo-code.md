# Longtable — API + React Query (compact)

Discovery → Details → Pay → Confirm | Pending → Recovery. Pseudo-code for the case study.

```ts
type VagueSlot = "brunch" | "lunch" | "dinner" | "evening" | "late";
type SeatingPrivacy = "any" | "large_table" | "semi_private" | "private_room";
type ConfirmMode = "instant" | "request_to_book";
type BookingStatus = "held" | "pending" | "confirmed" | "rejected" | "failed_availability";
type Money = { amount: number; currency: "GBP" };

/** Shareable URL = DiscoveryFilters in the route (client state). */
type Filters = {
  partySize: number;
  date: string; // YYYY-MM-DD
  vagueSlot: VagueSlot;
  seatingPrivacy: SeatingPrivacy;
  instantOnly?: boolean;
  vibe?: string;
};

type Recovery = {
  suggestions: Array<{ kind: "slot" | "time" | "venue"; label: string; restaurantId: string; roomId?: string }>;
};

type ApiError = {
  code: "SLOT_UNAVAILABLE" | "HOLD_EXPIRED" | "OWNER_CHANGED_INVENTORY";
  recovery?: Recovery;
};

const qk = {
  list: (f: Filters) => ["restaurants", f] as const,
  avail: (id: string, f: Omit<Filters, "instantOnly" | "vibe">) => ["availability", id, f] as const,
  booking: (id: string) => ["bookings", id] as const,
};
```

---



## Endpoints — URL + types + cache + invalidation in one place



### `GET /restaurants` — Discovery (available-only)

```ts
// GET /restaurants?...filters  →  { items, nextCursor?, staleAfterSec }
type ListItem = {
  id: string; name: string; cuisine: string; area: string;
  match: { partySize: number; seatingPrivacy: SeatingPrivacy };
  yourTime: { vagueSlot: VagueSlot; available: true };
  otherSlotsToday: VagueSlot[];
  otherDays: Array<{ date: string; vagueSlot: VagueSlot }>;
  priceFrom: Money;
};

function useList(filters: Filters) {
  // URL: /search?partySize&date&vagueSlot&seatingPrivacy&…  → filters
  return useQuery({
    queryKey: qk.list(filters), // filter change = new key (old cache kept briefly)
    queryFn: () => api.get("/restaurants", { params: filters }),
    staleTime: 30_000, // mild staleness OK (up to ~5min if staleAfterSec says so)
    gcTime: 5 * 60_000,
  });
}
```



### `GET /restaurants/:id/availability` — Details (concrete rooms)

```ts
// GET /restaurants/:id/availability?partySize&date&vagueSlot&seatingPrivacy
// → { rooms: BookableRoom[], staleAfterSec }
type BookableRoom = {
  roomId: string; name: string; seatMin: number; seatMax: number;
  window: { start: string; end: string }; // real inventory
  confirmMode: ConfirmMode; deposit: Money;
};

function useAvailability(id: string, f: Filters) {
  // URL: /restaurants/:id?…same primary filters
  const key = qk.avail(id, f);
  return useQuery({
    queryKey: key,
    queryFn: () => api.get(`/restaurants/${id}/availability`, { params: f }),
    staleTime: 15_000,           // tight TTL
    refetchInterval: 20_000,     // short poll (vs SSE)
    refetchOnWindowFocus: true,
  });
}
// On any Details filter change → invalidateQueries({ queryKey: ["availability", id] })
```



### `POST /bookings` — create (server rechecks; optional hold)

```ts
// POST /bookings  body → 201 CreateBooking | 409 ApiError+recovery
type CreateBody = {
  restaurantId: string; roomId: string; date: string;
  windowStart: string; windowEnd: string; partySize: number;
  seatingPrivacy: SeatingPrivacy; createHold?: boolean;
};
type CreateRes = {
  bookingId: string; status: "held"; confirmMode: ConfirmMode;
  holdExpiresAt?: string;ßß
};

function useCreateBooking() {
  const qc = useQueryClient();
  return useMutation({
    mutationFn: (body: CreateBody) => api.post("/bookings", body),
    onSettled: (_d, _e, body) => {
      qc.invalidateQueries({ queryKey: ["availability", body.restaurantId] });
      qc.invalidateQueries({ queryKey: ["restaurants"] });
    },
  });
}
```



### `POST /bookings/:id/pay` — re-validate then authorize

```ts
// POST /bookings/:id/pay  →  { status: "confirmed"|"pending" } | 409 ApiError+recovery
function usePay(bookingId: string, restaurantId: string) {
  const qc = useQueryClient();
  return useMutation({
    mutationFn: (paymentMethodId: string) =>
      api.post(`/bookings/${bookingId}/pay`, { paymentMethodId }),
    onSuccess: (res: { status: "confirmed" | "pending" }) => {
      // URL: /bookings/:id/confirmed | /bookings/:id/pending
      qc.setQueryData(qk.booking(bookingId), (b: any) => ({ ...b, status: res.status }));
      qc.invalidateQueries({ queryKey: ["availability", restaurantId] });
      qc.invalidateQueries({ queryKey: ["restaurants"] });
    },
    onError: () => {
      // Recovery route: /bookings/:id/recovery
      qc.invalidateQueries({ queryKey: qk.booking(bookingId) });
      qc.invalidateQueries({ queryKey: ["availability", restaurantId] });
    },
  });
}
```



### `GET /bookings/:id` — Pending / Confirm / Rejected

```ts
// GET /bookings/:id  →  BookingDetail (status drives screen)
type BookingDetail = {
  bookingId: string; status: BookingStatus; confirmMode: ConfirmMode;
  restaurant: { id: string; name: string; phone?: string };
  room: { name: string; window: { start: string; end: string } };
  avgResponseMinutes?: number; // pending UX
  shortcuts?: { inviteTemplateId?: string; mapsUrl?: string };
  recovery?: Recovery;
};

function useBooking(id: string) {
  // URL: /bookings/:id
  return useQuery({
    queryKey: qk.booking(id),
    queryFn: () => api.get(`/bookings/${id}`),
    staleTime: 5_000,
    refetchInterval: (q) =>
      ["pending", "held"].includes(q.state.data?.status) ? 5_000 : false,
  });
}
```

---



## Policy in one glance


| Resource     | URL                                               | Cache                    | Invalidate when                            |
| ------------ | ------------------------------------------------- | ------------------------ | ------------------------------------------ |
| List         | `/search?filters` → `GET /restaurants`            | `staleTime` 30s–5m       | write booking / pay; new filters = new key |
| Availability | `/restaurants/:id?filters` → `GET …/availability` | 15s + poll 20s           | filter change; create; pay; 409            |
| Booking      | `/bookings/:id`                                   | 5s; poll if pending/held | pay success/fail; status transitions       |


Server state = list / availability / booking. Client+URL = filters, selected `roomId`, guest-list draft. Writes always recheck on server; 409 → Recovery, never a dead end. DB unique `(restaurant, room/slot, date)` is the double-book backstop.