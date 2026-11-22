# Dual-Axis Off-Grid Solar Tracker (ESP32)

Sistem autonom de urmărire solară pe două axe (azimut și elevație), proiectat pentru maximizarea randamentului unui panou fotovoltaic de mici dimensiuni. Proiectul combină controlul electromecanic în buclă închisă, monitorizarea energetică în timp real și o arhitectură de alimentare complet off-grid, fiind integrat într-o structură modulară printată 3D.

---

## Funcționalități Principale

* **Urmărire automată pe 2 axe:** Orientare continuă către sursa de lumină folosind o matrice de 4 fotorezistoare (LDR) dispuse în cadrane (Stânga-Sus, Dreapta-Sus, Stânga-Jos, Dreapta-Jos).
* **Monitorizare energetică în timp real:** Măsurarea precisă a tensiunii (V), curentului (mA) și puterii (mW) prin senzorul dedicat INA226 (shunt `0.1Ω`), cu afișare locală pe un ecran LCD 1602.
* **Optimizare pentru consum redus:**
  * *Zonă moartă (Deadzone):* Elimină oscilațiile și mișcările inutile ale motoarelor la variații minore de lumină.
  * *Deconectarea bobinelor (Coil Release):* Dezactivează pinii motoarelor pas cu pas în repaus pentru a preveni consumul pasiv și supraîncălzirea.
* **Alimentare Off-Grid:** Sistem autonom bazat pe panou solar, modul de încărcare cu protecție (TP4056), acumulator Li-Ion 18650 și convertor ridicător de tensiune (MT3608) calibrat la 5.0V.
* **Design mecanic modular:** Mecanism pan-tilt printat 3D separat de incinta electronică, facilitând mentenanța și extinderea.

---

## Componente Hardware (BOM)

| Componentă | Cantitate | Rol în sistem |
| :--- | :---: | :--- |
| **ESP32 DevKit V1** | 1 | Unitate centrală de control, achiziție ADC, comunicație I2C și WiFi |
| **Motor pas cu pas 28BYJ-48 (5V)** | 2 | Acționare mecanică pe axa orizontală (H) și verticală (V) |
| **Driver ULN2003** | 2 | Etaj de putere pentru comanda motoarelor pas cu pas |
| **Fotorezistor (LDR) + Rezistență 10kΩ** | 4 | Divizoare de tensiune (3.3V) pentru detecția direcției luminii |
| **Senzor INA226 (I2C, shunt R100)** | 1 | Monitorizare tensiune, curent și putere pe magistrala solară |
| **Ecran LCD 1602 + adaptor I2C** | 1 | Afișare locală a parametrilor energetici |
| **Panou solar fotovoltaic** | 1 | Captarea energiei solare și încărcarea acumulatorului |
| **Modul încărcare TP4056 (cu protecție)** | 1 | Managementul încărcării și protecția bateriei 18650 |
| **Acumulator Li-Ion 18650** | 1 | Stocarea energiei pentru operare off-grid |
| **Modul Step-Up MT3608** | 1 | Ridicarea și stabilizarea tensiunii de la 3.7V la 5.0V |

---

## Arhitectura Electrică & Pinout

### 1. Conexiuni ESP32

| Modul / Senzor | Pin Modul | Pin ESP32 | Note |
| :--- | :--- | :--- | :--- |
| **LDR Stânga-Sus (TL)** | Semnal | `D32` | Divizor tensiune pe `3V3` |
| **LDR Dreapta-Sus (TR)** | Semnal | `D33` | Divizor tensiune pe `3V3` |
| **LDR Stânga-Jos (BL)** | Semnal | `D34` | Divizor tensiune pe `3V3` |
| **LDR Dreapta-Jos (BR)** | Semnal | `D35` | Divizor tensiune pe `3V3` |
| **Motor Orizontal (ULN2003)** | IN1, IN2, IN3, IN4 | `D19, D18, D5, D17` | Alimentare 5V |
| **Motor Vertical (ULN2003)** | IN1, IN2, IN3, IN4 | `D27, D26, D25, D14` | Alimentare 5V |
| **Magistrala I2C (LCD & INA226)** | SDA / SCL | `D21` / `D22` | Adrese: LCD `0x27`, INA226 `0x40` |

### 2. Fluxul de Alimentare Off-Grid

```text
[Panou Solar] ──► [INA226 (IN+/IN-)] ──► [TP4056 (IN+/IN-)]
                                                │
                                         [Baterie 18650]
                                                │
                                         [TP4056 (OUT)]
                                                │
                                     [MT3608 Step-Up (5.0V)]
                                                │
                             ┌──────────────────┼──────────────────┐
                             ▼                  ▼                  ▼
                      [ESP32 (VIN)]       [LCD 1602 (5V)]   [Drivere ULN2003]
```

*(Senzorii LDR și modulul INA226 sunt alimentați separat din regulatorul intern `3V3` al plăcii ESP32, având masă comună `GND` cu restul circuitului.)*

---

## Arhitectura Software

Firmware-ul este scris în C++ (Arduino Framework pentru ESP32) și utilizează următoarele biblioteci:
* `Wire.h` – Comunicație I2C
* `Stepper.h` – Controlul motoarelor 28BYJ-48 (2048 pași/rotație)
* `LiquidCrystal_I2C` *(Frank de Brabander)* – Control LCD 1602
* `INA226` *(Rob Tillaart)* – Citirea și calibrarea senzorului de curent/tensiune

### Algoritmul de urmărire
1. Se citesc valorile analogice pe 12 biți (`0–4095`) de la cei 4 senzori LDR.
2. Se calculează mediile pe cele 4 direcții: `avgTop`, `avgBottom`, `avgLeft`, `avgRight`.
3. Se determină erorile diferențiale `diffV = avgTop - avgBottom` și `diffH = avgLeft - avgRight`.
4. Dacă eroarea depășește pragul `DEADZONE` (150), motoarele execută pași incrementali (`STEP_CHUNK = 20`) spre zona cu intensitate luminoasă mai mare.
5. Când ambele axe se încadrează în zona moartă, funcția `releaseMotors()` pune toți pinii motoarelor în `LOW` pentru economisirea bateriei.
6. La fiecare 1000 ms (non-blocking, folosind `millis()`), se interoghează INA226 și se actualizează afișajul LCD.

---

## Structura Proiectului

```text
solar-tracker-esp32/
├── firmware/
│   ├── solar_tracker/          # Codul principal (tracking + telemetrie)
│   └── tests/                  # Scripturi de test individuale (I2C scanner, LDR, motoare)
├── cad/                        # Modele 3D (.stl) pentru mecanism și carcasă
├── docs/                       # Scheme electrice și poze din timpul asamblării
└── README.md
```

---

## Stadiul Proiectului (Roadmap)

- [x] Testare individuală motoare 28BYJ-48 + drivere ULN2003
- [x] Configurare și calibrare matrice 4x LDR (3.3V)
- [x] Integrare și validare magistrală I2C (LCD `0x27` + INA226 `0x40`)
- [x] Implementare firmware de bază (tracking automat + afișare live + coil release)
- [ ] Finalizare asamblare mecanică 3D și adăugare limite software de rotație
- [ ] Integrare completă alimentare off-grid (suport 18650 + calibrare MT3608 la 5.0V)
- [ ] Interfață Web WiFi (ESP32 Web Server) pentru monitorizare live și control manual/auto
