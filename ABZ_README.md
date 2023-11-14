# ABZ Innovation - Szoftverfejlesztői szakmai interjú teszt feladat (Gaál Csaba)

## Feladat leírása
> A firmware
> ([https://github.com/ArduPilot/ardupilot](https://github.com/ArduPilot/ardupilot))
> küldjön vissza másodpercenként 1 üzenetet, hogy éppen hol tart a
> missionben, amennyiben nincsen feltöltve rá mission, akkor küldje
> vissza hogy nincsen mission.   A 4.3.2-es verziójú ArduCopter Tag-en
> dolgozzon.   Ezen felül hozzon létre egy új Mavlink message-t 1500-as
> ID-val. A mission indításakor küldje el, úgy, hogy az 1. paraméter
> értéke 1.

## Feladat megoldása

### ArduPilot (ArduCopter 4.3.2) projekt forkolása

A fork a [https://github.com/megariffer/ardupilot-abz](https://github.com/megariffer/ardupilot-abz) URL-en elérhető publikus repoban található.
A fejlesztést az [abz-task](https://github.com/megariffer/ardupilot-abz/tree/abz-task) branchen vezettem.

A branch klónozása:
```
git clone -b abz-test git@github.com:megariffer/ardupilot-abz.git
```

### Üzenet küldése minden másodpercben

Másodpercenkénti üzenetek küldésére (vagy bármilyen 1Hz-es gyakorisággal végzett műveletre) a Copter.cpp-ben található `Copter::one_hz_loop()` metódus a legegyszerűbb megoldás.

- Felhasznált dokumentáció: https://ardupilot.org/dev/docs/code-overview-scheduling-your-new-code-to-run-intermittently.html#
- Megvalósítás: https://github.com/megariffer/ardupilot-abz/commit/0ff0cf6346b5b0fd9cfb0081bbb4df4be49d2050

Itt mindössze egy saját metódust kell meghívnunk, amelyben megvalósítjuk az üzenetküldést a [GCS singleton segítségével](https://ardupilot.org/dev/docs/debugging.html#mavlink-messages).

Hogy hol tartunk a missionben, azt két egyszerű paranccsal ki tudjuk számolni:
`mode_auto.mission.num_commands()` amely a missionben tárolt összes command számát adja meg, illetve a `mode_auto.mission.get_current_nav_index()`, amely pedig az éppen aktuális command indexét adja vissza.
Egyéb hasznos információkat a drón aktuális helyzetéről (GPS koordináták, magasság) a `current_loc` fieldben találunk.

**A részfeladat tesztelése:**

Tesztelés a SITL programban történik.
A mission tesztelése a repoban található myway.txt mission fájllal történik. 
Ez egy egyszerű, 10 waypointból álló mission Tahitótfalu felett (ezért a `sim_vehicle.py` szkriptnek ezeket a koordinátákat adjuk meg).

```
cd ardupilot-abz/ArduCopter
../Tools/autotest/sim_vehicle.py --console --map -l 47.74224205318202,19.069562424870714,0,0
```
A build után elindul a SITL program. A drónt helyezzük GUIDED üzemmódba, élesítsük, és szálljunk fel vele (példa kedvéért 10m magasságba). Ehhez a terminálban adjuk ki ezeket a parancsokat:
```
mode guided
arm throttle
takeoff 10
```
Következő lépésben a SITL programban töltsük be a missiont (SITL consoleban Mission/Load -> myway.txt).

Ezután indítható a mission AUTOPILOT üzemmódban:
```
mode auto
```
Ha minden sikeres, akkor a következő formátumban fognak érkezni az üzenetek a SITL konzolba:
>AP: Lat: 47.74224, Lng: 19.06960, Alt: 12. Completed 10% of mission (2/10 commands).

Mission törlése után (SITL konzolban Mission/Clear) a következő üzenetet küldi a firmware:
>AP: No waypoints are loaded.

### Egyedi MAVLink message létrehozása

Felhasznált dokumentáció: https://ardupilot.org/dev/docs/code-overview-adding-a-new-mavlink-message.html

MAVLink üzenet létrehozásához első lépésként a mgfelelő helyen létre kell hoznunk az üzenet szerkezetét XML formátumban.
Erre a megfelelő hely a `modules/mavlink/message_definitions/v1.0/ardupilotmega.xml` fájl, ahol az XML fájl végére bemásoljuk az üzenetünket:
```
<message id="1500" name="ABZ_TEST_MESSAGE">
    <description>A test message that could possibly land (pun intended) a job for its developer.</description>
    <field type="uint8_t" name="first">The first (and only) param.</field>
</message>
```

Ha ezzel megvagyunk, egy `./waf copter` paranccsal le kell buildelnünk a projektet, vagy ez a `sim_vehicle.py` szkript indításával is megtörténik.

Ezzel elkészül üzenetünk header fájlja a `build/sitl/libraries/GCS_MAVLink/include/mavlink/v2.0/ardupilotmega/mavlink_msg_abz_test_message.h` útvonalon.

Ahhoz, hogy az üzenetet használni tudjuk, fel kell vennünk az azonosítóját a GCS Message ID-k közé. Ezt az `ap_message.h` tudjuk megtenni, az `enum ap_message` enum kibővítésével.

Magát a küldést a GCS interface-ben tudjuk megvalósítani. Ehhez létre kell hozni a megfelelő definíciót a `libraries/GCS_MAVLink/GCS.h` fájlban. Ebben a feladatban ezt a `GCS_MAVLINK::send_abz_test()` metódusban definiáltam.
A metódus megvalósítása a `libraries/GCS_MAVLink/GCS_Common.cpp` fájlban történik, ahol a generált header fájl `mavlink_msg_abz_test_message_send()` metódusát hívjuk meg a megfelelő paraméterekkel.

Az üzenet küldéséhez szükség van még ugyanezen fájl `GCS_MAVLINK::try_send_message()` metódusában lekezelni az üzenetet. Ezt az itt lévő switch case kibővítésével tudjuk megtenni. A case vizsgált értéke az `ap_message.h` fájlban korábban létrehozott GCS Message ID-t tartalmazza. Itt hívjuk meg az előbb létrehozott `send_abz_test()` metódust.

Ezzel az üzenetünk készen áll a használatra. A tényleges GCS oldali kezelése (tehát hogy például kiírjuk az üzenetben kapott mező értékét) nem a feladat része, így azt itt nem valósítjuk meg.

### Végszó

Köszönöm a lehetőséget Lázár Tamásnak és Ludvigh Károlynak, illetve az ABZ Innovationnek. Izgalmas betekintés volt a drónok világába 😊

Gaál Csaba, 2023.11.14.