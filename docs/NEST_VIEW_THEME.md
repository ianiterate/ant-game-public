# Nest view — theme

What the animated nest cutaway looks like and why. The rules it shows (which count drives which
animation, how many figures stand for how many workers) are in [NEST_VIEW.md](NEST_VIEW.md); the
words are in [UI_COPY.md](UI_COPY.md#nest-view). This file is the place, not the mechanism.

## The conceit

The panel is a section cut through the mound, as if a spade had gone in beside the entrance and
the soil had been lifted away. The cut face is the frame: dark umber grains, hair roots sliced
through, a pebble cut in half. Inside it the colony is going about its business, and none of it
knows it is being watched.

Three rules hold everywhere:

- **Every motion is a count or an event.** Nurses on screen are `Nursing`, diggers are `Crew`, an
  egg appears because the queen laid one. Nothing busy-looks for its own sake. If the view shows
  work, the sim is doing it.
- **The normal state needs no label.** An egg being laid says "laying". Labels appear only where
  something is wrong or about to change, so a label always means *look here*.
- **Naturalistic, never magical.** No lamps, no glow, no particles of light. The nest is lit the
  way a cut-open bank of soil would be lit: from the open side, falling off with depth.

**Assumed** — the spade-cut framing and the three rules. Cheap to change: art direction only.

## The places

**The queen's chamber.** The deepest, widest room, drawn first. She is twice a worker's length,
the same dark chitin, with a long heavy gaster that has an amber-brown sheen where the light
catches it and two short stubs where wings once were. She does not walk. Two or three attendants
stand at her head and flank, antennae on her, grooming. When she lays, the gaster heaves once, a
pale egg is there, and an attendant picks it up and turns away up the tunnel towards the brood. A
small clump of eggs and a few grains of food sit by her (the chamber "holds a little brood and
food"). *Blocked*: she is still, the attendants are still, and the brood chambers above are packed
to their doorways — the eye goes from her to the crowded door and understands. *Not laying* (no food,
or winter): she is still, but the brood doors are open and the store chamber is bare.

**Brood chambers.** Brood is sorted by age in three clusters along the floor: eggs (a sticky ivory
clump), larvae (curled cream grubs, a grey line showing through), pupae (straw-coloured cocoons in a
neat row nearest the door). Nurses move constantly and slowly: lift a larva, turn it, set it down,
bend to one and feed it, carry a cocoon a body length closer to the door as it ages. At midnight the
day's hatch happens here: a cocoon splits, a pale worker stands, darkens to the colony's brown over a
few seconds, and walks out to the landing. *Too few nurses*: clusters lie untouched and dull, and now
and then a nurse carries a still larva out towards the midden.

**Assumed** — brood age maps to stage in thirds of the seven days (eggs days 1–2, larvae 3–5, pupae
6–7), and pupae are cocooned. Cheap to change: presentation only. Strings say "brood", never larva
or pupa (UI_COPY.md voice: no biology the sim does not model).

**Store chambers.** One pile per chamber, its height the chamber's fill. The stores are what came in:
sugar as loose white grains that slide and settle, seeds stacked like hard tan pebbles, dry leaf cut
into ochre squares laid flat, and dark insect pieces — leg segments, a curved shell plate, pale meat
at the cut. Workers arrive, tip their load on the pile, and the top slumps a little. *Full*: the pile
reaches the doorway and workers stand in the tunnel holding loads. *Overflow*: grains spill out of
the door and down the tunnel floor, where they are left — "lost at the door". *Empty*: bare swept
floor and a few husks. Stored food does not spoil, so the pile never rots.

**Assumed** — the top layer of a pile shows the kinds most recently delivered. The sim keeps food
as one number, so the view must remember recent deliveries itself. Cheap; view-side only.

**The processing chamber — the cutting room.** A dead insect is too big to store whole and rots in
three days where it lies; the processing chamber is where it is taken apart into pieces that keep.
When one is hauled home it is dragged in whole, on its back, and the workers cut it at the joints:
legs off, head off, shell pried from the body, the soft parts portioned and carried away to the
stores. The shell plates go out to the midden. Between insects the room is swept and empty, a
couple of workers scraping the floor, the last plates waiting by the door. This is why the chamber
gates dead insects and nothing else: seeds, leaves and sugar already come in store-sized.

**Assumed** — processing is the cutting room. This is the first time the fiction says what processing
*is*. Cheap to change: one string (`nestview.processing.cutting`) and the chamber's art. If processing
ever gains a mechanic (seed cracking, leaf paste), this concept extends to it without renaming.

**Tunnels, the landing and the midden.** One shaft runs down from the entrance at the top of the
frame; side tunnels branch to the chambers. Just below the entrance is **the landing**, a wide gallery
where idle workers wait — its crowd is the idle count, and it is where trails and taps draw from. Off
the shaft near the surface is **the midden**, a dead-end pocket of husks, shell plates and the dead.

**Assumed** — the landing and the midden. Neither is a chamber in the sim. Cheap: art only.

**A dig in progress.** The slot shows a rough tunnel stub with a crumbling face. Diggers scrape at
the face, roll the loose soil into a pellet in their jaws, climb the shaft, and drop it over the rim
out of the top of the frame. The cavity grows as the dig progresses; the floor is darker and damper
than finished chambers. *Waiting for idle workers*: the stub is there, the face is still, a pellet lies
where it was dropped. Undug slots are solid soil with roots; locked slots are deeper, denser soil with
no stub at all.

**Day and night.** By day a warm wedge of light comes down the shaft and the upper chambers sit in it;
deeper rooms fall off to brown dark. At night the wedge turns low and blue-grey, the whole section
drops a stop, and the surface sounds go quiet. The work goes on — digging and nursing do not stop in
the sim — but the landing settles: idle workers fold their legs and stop shuffling.

**Assumed** — night changes the light, not the work. Cheap to change.

**Winter.** The cold comes in from the top. A pale frost line creeps down the cut face from the
surface; how far it reaches is the thatch. Thatch is drawn as pinecone scales layered over the
entrance; each one closes a gap, and at full thatch the frost stops at the scales. The colony
huddles: the landing empties, everyone packs into the queen's chamber around her in a tight dark
mass, and the only motion is a slow shift at the edges.

**Assumed** — frost depth shows `Shelter`, and the winter huddle is in the queen's chamber. Cheap.

**Starvation.** The store floors are bare. Workers are thin — gasters narrow where they were round
— and slow: gaps open between them, they stop mid-step, the landing is sparse. The queen is still.
About one in ten a day, a worker is carried up to the midden.

**Raids.** Raiders are the rival colony's red-brown (`Ant_Rival.mat`), the colony's own dark brown
against them. *Marching*: the landing packs tight below the entrance, every head towards the top.
*Fighting*: defenders jam the mouth of the shaft, grappling raiders in the entrance; while you stand
at the nest your ant is among them, picked out by its cool rim. *Breach*: raiders pour down the shaft
to the store chambers and carry pieces back up and out; the piles drop. Nurses close over the brood
clusters, but the queen and the brood are never touched.

## Palette

Shared soil: cut face `#5A4130`, deep soil `#2E2119`, cut-face rim `#2A1E16`, hair roots `#C2A27A`.
Workers `#1F140E` (`Ant_Body.mat`), raiders `#6B2917`.

| Place | Palette |
|---|---|
| Queen's chamber | floor `#6B4A32`, wall `#4A3324`, queen `#1A100B`, gaster sheen `#7A5230`, egg `#EFE8D6` |
| Brood | floor `#6E5038` (swept, lighter), eggs `#EFE8D6`, larvae `#E3D6BC`, gut line `#9C8F7A`, cocoons `#C9AE84` |
| Store | sugar `#F4F1EA`, seeds `#8C6A3E` / husk `#B08D57`, leaf `#8A6A2E`, shell `#2B211C`, meat `#B98A55`, floor `#5E4532` |
| Processing | carcass `#1E2226` with sheen `#2F3A33`, cut joints `#C8B48C`, stained floor `#4A3527` |
| Midden | husks and plates `#6B6258` over `#3A2E25` |
| Dig | fresh face `#3E2B1E`, pellets `#4D3626`, mound top `#8A6E52` |
| Light | day key `#FFE9C4` falling to `#1A130E`; night key `#9FB2CC` falling to `#0E0D10` |
| Winter | frost `#D8E0E6`; whole section desaturated about 25% |
| Starvation | stores bare to floor colour; whole section desaturated about 30% |

## Motion verbs

| Place | Verbs |
|---|---|
| Queen's chamber | heave, lay, groom, carry off, settle |
| Brood | turn, feed, shift (by age), carry, hatch |
| Store | tip, slump, stack, spill, queue |
| Processing | drag, cut, pry, portion, haul out |
| Dig | scrape, roll, climb, drop, crumble |
| Landing (night) | fold, settle |
| Winter | huddle, pack, shift |
| Starvation | falter, stall, carry up |
| Raid | pack, grapple, push, pour in, drag off |
