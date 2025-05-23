<script lang="ts">
    import { onMount } from 'svelte';
    import { browser } from '$app/environment';
    import { page } from '$app/stores';

    // Typen definieren
    type Koordinate = {
        lat: number;
        lng: number;
    };

    type Station = {
        id: number;
        name: string;
        koordinate: Koordinate;
        tipp: string;
        raetsel: string;
        loesung: string;
    };

    // Typ für gespeicherten Fortschritt
    type SpielFortschritt = {
        aktuelleStationIndex: number;
        geloesteListe: number[];
        spielGestartet: boolean;
    };

    // Konstante für den localStorage Key
    const STORAGE_KEY = 'schnitzeljagd_fortschritt';

    // Stationen der Schnitzeljagd
    const stationen: Station[] = [
        {
            id: 1,
            name: "Erste Station",
            koordinate: { lat: 52.57337, lng: 13.50440 }, // Beispiel: watenberg kordinaten
            tipp: "Habt ihr euer transport mittel genau unter die lupe genommen?",
            raetsel: "Worauf Autofahrer im straßenverker achten?",
            loesung: "schilder"
        },
        {
            id: 2,
            name: "Zweite Station",
            koordinate: { lat: 52.50494444, lng: 13.45319444 }, // Etwas weiter nordöstlich
            tipp: "Ich glaube ihr solltet diese genauer betrachten!",
            raetsel: "Skate board Rätzel",
            loesung: "skateboard"
        },
        {
            id: 3,
            name: "Dritte Station",
            koordinate: { lat: 52.50702778, lng: 13.45413889 }, // Noch weiter nordöstlich
            tipp: "Wo kann ich fahren? Zeigt an der Teke der Halle euer Gruppensymbol vor.",
            raetsel: "Um über die stadt zu sehen muss man mich erklimmen was bin ich?",
            loesung: "berg"
        },
        {
            id: 4,
            name: "Vierte Station",
            koordinate: { lat: 52.52638889, lng: 13.43202778 }, // Noch weiter nordöstlich
            tipp: "Finde den baum mit der nummer aus 27.540 durch 4 geteilt und schaut hinter die mauer.",
            raetsel: "Ich schmelze an heißen Tagen. Was bin ich?",
            loesung: "eis"
        },
        {
            id: 5,
            name: "Fünfte Station",
            koordinate: { lat: 52.520083, lng: 13.411139 }, // Noch weiter nordöstlich
            tipp: "Sprecht mit dem Softinis-Mitarbeiter und zeigt ihm euer gruppen symbol!",
            raetsel: "Auf mir fahren Züge was bin ich?",
            loesung: "schiene"
        },
        {
            id: 6,
            name: "Sechste Station",
            koordinate: { lat: 52.4971108, lng: 13.3724448 }, // Noch weiter nordöstlich
            tipp: "Ich bin ein kasten und versteck mich im grünen!",
            raetsel: "flughafen rätzel",
            loesung: "flughafen"
        },
        {
            id: 7,
            name: "Siebte Station",
            koordinate: { lat: 52.47238889, lng: 13.3895 }, // Noch weiter nordöstlich
            tipp: "Ich brauche ein vortbewegungs mittel.",
            raetsel: "Findet uns am ender der Landebahn um die Jagt zu benden",
            loesung: "bruno"
        },
{
            id: 8,
            name: "Achte Station",
            koordinate: { lat: 52.50589623, lng: 13.44774266 }, // Noch weiter nordöstlich
            tipp: "Geht zum Peter Pan!",
            raetsel: "Was gibt es denn hier zu essen?",
            loesung: "burger"
        }
    ];

    // Status Variablen
    let aktuellePosition: Koordinate | null = null;
    let entfernung: number = -1;
    let aktuelleStationIndex: number = 0;
    let aktuelleStation: Station = stationen[aktuelleStationIndex];
    let benutzerAntwort: string = "";
    let fehlerNachricht: string = "";
    let erfolgNachricht: string = "";
    let tippAnzeigen: boolean = false;
    let geloesteListe: number[] = [];
    let positionGefunden: boolean = false;
    let positionError: string = "";
    let spielBeendet: boolean = false;
    let spielGestartet: boolean = false;
    let watchId: number | null = null;
    let statusNachricht: string = "Tippe auf 'Spiel starten', um zu beginnen";
    let hatGespeichertenFortschritt: boolean = false;

    // Karten-Variablen
    let map: any = null;
    let positionsMarker: any = null;
    let radiusKreis: any = null;
    let stationsMarker: any = null;

    // NEU: Variable für automatisches Zentrieren
    let autoZentrieren: boolean = true;
    let standortAktualisieren: boolean = false;

    // TEST-MODUS Variablen
    let testModus: boolean = false;
    let testPosition: Koordinate = { lat: 52.520000, lng: 13.404950 }; // Nahe der ersten Station
    let bewegungsVektor: Koordinate = { lat: 0.00001, lng: 0.00001 }; // Für die Simulation der Bewegung
    let debugInfo: string = "";

    // Debug-Einstellungen
    let suchRadius: number = 100; // in Metern
    let simulationsGeschwindigkeit: number = 2000; // in Millisekunden

    // Fortschritt speichern
    function speichereFortschritt() {
        if (browser) {
            const fortschritt: SpielFortschritt = {
                aktuelleStationIndex,
                geloesteListe,
                spielGestartet
            };

            try {
                localStorage.setItem(STORAGE_KEY, JSON.stringify(fortschritt));
                console.log("Fortschritt gespeichert:", fortschritt);
            } catch (error) {
                console.error("Fehler beim Speichern des Fortschritts:", error);
            }
        }
    }

    // Fortschritt laden
    function ladeFortschritt(): boolean {
        if (browser) {
            try {
                const gespeicherterFortschritt = localStorage.getItem(STORAGE_KEY);

                if (gespeicherterFortschritt) {
                    const fortschritt: SpielFortschritt = JSON.parse(gespeicherterFortschritt);

                    // Nur laden, wenn das Spiel nicht beendet ist
                    if (fortschritt.aktuelleStationIndex < stationen.length) {
                        aktuelleStationIndex = fortschritt.aktuelleStationIndex;
                        geloesteListe = fortschritt.geloesteListe;
                        // WICHTIG: Hier spielGestartet nicht setzen, sonst wird die UI falsch angezeigt
                        // Wir setzen spielGestartet erst bei setzeSpielFort()
                        aktuelleStation = stationen[aktuelleStationIndex];

                        // Prüfen, ob das Spiel tatsächlich beendet ist (alle Stationen gelöst)
                        if (aktuelleStationIndex === stationen.length - 1 &&
                            geloesteListe.includes(stationen[stationen.length - 1].id)) {
                            spielBeendet = true;
                        }

                        console.log("Fortschritt geladen:", fortschritt);
                        return true;
                    }
                }
            } catch (error) {
                console.error("Fehler beim Laden des Fortschritts:", error);
            }
        }

        return false;
    }

    // Fortschritt zurücksetzen
    function loescheFortschritt() {
        if (browser) {
            try {
                localStorage.removeItem(STORAGE_KEY);
                console.log("Fortschritt zurückgesetzt");
            } catch (error) {
                console.error("Fehler beim Zurücksetzen des Fortschritts:", error);
            }
        }
    }

    // Entfernung zwischen zwei Koordinaten in Metern berechnen (Haversine-Formel)
    function berechneEntfernung(start: Koordinate, ziel: Koordinate): number {
        const R = 6371e3; // Erdradius in Metern
        const phi1 = start.lat * Math.PI / 180;
        const phi2 = ziel.lat * Math.PI / 180;
        const deltaPhi = (ziel.lat - start.lat) * Math.PI / 180;
        const deltaLambda = (ziel.lng - start.lng) * Math.PI / 180;

        const a = Math.sin(deltaPhi / 2) * Math.sin(deltaPhi / 2) +
            Math.cos(phi1) * Math.cos(phi2) *
            Math.sin(deltaLambda / 2) * Math.sin(deltaLambda / 2);
        const c = 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1 - a));

        return R * c;
    }

    // Karte initialisieren
    function initialisiereKarte(position: Koordinate) {
        if (!browser) return;

        // Leaflet-Skript und CSS laden, falls noch nicht geschehen
        if (!document.getElementById('leaflet-css')) {
            const link = document.createElement('link');
            link.id = 'leaflet-css';
            link.rel = 'stylesheet';
            link.href = 'https://unpkg.com/leaflet@1.9.4/dist/leaflet.css';
            document.head.appendChild(link);
        }

        if (!window.L && !document.getElementById('leaflet-script')) {
            const script = document.createElement('script');
            script.id = 'leaflet-script';
            script.src = 'https://unpkg.com/leaflet@1.9.4/dist/leaflet.js';
            script.onload = () => {
                console.log("Leaflet geladen, erstelle Karte");
                erstelleKarte(position);
            };
            document.head.appendChild(script);
        } else if (window.L) {
            console.log("Leaflet bereits geladen, erstelle Karte");
            erstelleKarte(position);
        }
    }

    // Karte erstellen
    function erstelleKarte(position: Koordinate) {
        if (!window.L) {
            console.error("Leaflet ist nicht verfügbar");
            return;
        }

        // Container für die Karte überprüfen und bereinigen
        const mapContainer = document.getElementById('map');
        if (!mapContainer) {
            console.error("Map-Container nicht gefunden");
            return;
        }

        mapContainer.innerHTML = '';

        try {
            console.log("Erstelle Karte mit Position:", position);

            // Karte erstellen mit Vollbild-Stil - dunkler Stil für Uber-Look
            map = L.map('map', {
                zoomControl: false, // Entferne Standard-Zoom-Steuerelemente
                attributionControl: false // Entferne Standard-Attribution
            }).setView([position.lat, position.lng], 16);

            // Zoom-Steuerelemente an die untere rechte Ecke verschieben
            L.control.zoom({
                position: 'bottomright'
            }).addTo(map);

            // Dunkles Kartenhema für Uber-Look verwenden - CartoDB Dark Matter
            L.tileLayer('https://{s}.basemaps.cartocdn.com/dark_all/{z}/{x}/{y}{r}.png', {
                maxZoom: 19,
                attribution: '&copy; <a href="https://www.openstreetmap.org/copyright">OpenStreetMap</a>, &copy; <a href="https://carto.com/attributions">CARTO</a>'
            }).addTo(map);

            // Positionsmarker hinzufügen - Blauer Punkt im Uber-Stil
            positionsMarker = L.marker([position.lat, position.lng], {
                icon: L.divIcon({
                    className: 'position-marker',
                    html: '<div class="w-4 h-4 bg-blue-500 rounded-full border-2 border-white shadow-lg"></div>',
                    iconSize: [20, 20],
                    iconAnchor: [10, 10]
                })
            }).addTo(map);

            // Stationsmarker hinzufügen - Roter Zielpunkt im Uber-Stil
            stationsMarker = L.marker([aktuelleStation.koordinate.lat, aktuelleStation.koordinate.lng], {
                icon: L.divIcon({
                    className: 'station-marker',
                    html: '<div class="w-6 h-6 bg-red-500 border-2 border-white rounded-full flex items-center justify-center text-white font-bold text-xs">📍</div>',
                    iconSize: [24, 24],
                    iconAnchor: [12, 12]
                })
            }).addTo(map);

            // 100m-Radius um die Station zeichnen - Subtle für Uber-Style
            radiusKreis = L.circle([aktuelleStation.koordinate.lat, aktuelleStation.koordinate.lng], {
                radius: suchRadius,
                color: '#333',
                fillColor: '#555',
                fillOpacity: 0.2,
                weight: 1
            }).addTo(map);

            // Karte auf beide Marker zentrieren
            const gruppe = L.featureGroup([positionsMarker, stationsMarker]);
            map.fitBounds(gruppe.getBounds().pad(0.3));

            console.log("Karte erfolgreich erstellt");
        } catch (error) {
            console.error("Fehler beim Erstellen der Karte:", error);
        }
    }

    // Karte aktualisieren - GEÄNDERT: nur zentrieren wenn autoZentrieren aktiv ist
    function aktualisiereKarte() {
        if (!map || !aktuellePosition) return;

        // Positionsmarker aktualisieren
        positionsMarker.setLatLng([aktuellePosition.lat, aktuellePosition.lng]);

        // Stationsmarker und Radius aktualisieren
        stationsMarker.setLatLng([aktuelleStation.koordinate.lat, aktuelleStation.koordinate.lng]);
        radiusKreis.setLatLng([aktuelleStation.koordinate.lat, aktuelleStation.koordinate.lng]);
        radiusKreis.setRadius(suchRadius);

        // Karte nur zentrieren wenn autoZentrieren aktiv ist oder explizit angefordert
        if (autoZentrieren || standortAktualisieren) {
            const bounds = map.getBounds();
            if (!bounds.contains(positionsMarker.getLatLng()) || !bounds.contains(stationsMarker.getLatLng())) {
                const gruppe = L.featureGroup([positionsMarker, stationsMarker]);
                map.fitBounds(gruppe.getBounds().pad(0.3));
            }
            standortAktualisieren = false; // Reset nach manueller Aktualisierung
        }
    }

    // NEU: Manuell Standort aktualisieren
    function aktualisiereStandortManuell() {
        if (testModus) {
            // Im Test-Modus: Position zur Station bewegen
            testPosition = {
                lat: aktuelleStation.koordinate.lat + (Math.random() - 0.5) * 0.001,
                lng: aktuelleStation.koordinate.lng + (Math.random() - 0.5) * 0.001
            };
            standortAktualisieren = true;
            aktualisiereTestPosition();
        } else if (browser && navigator.geolocation) {
            // Echte Geolocation verwenden
            standortAktualisieren = true;
            navigator.geolocation.getCurrentPosition(
                (position) => {
                    console.log("Manueller Standort erhalten:", position.coords);
                    aktualisierePosition(position);
                },
                (error) => {
                    console.error("Fehler bei manueller Standortabfrage:", error);
                    positionError = "Standort konnte nicht aktualisiert werden.";
                },
                { enableHighAccuracy: true, maximumAge: 0, timeout: 10000 }
            );
        }
    }

    // NEU: Karte zu aktueller Position zentrieren
    function zentriereAufPosition() {
        if (map && aktuellePosition) {
            map.setView([aktuellePosition.lat, aktuellePosition.lng], 18, { animate: true });
        }
    }

    // NEU: Karte zu Station zentrieren
    function zentriereAufStation() {
        if (map && aktuelleStation) {
            map.setView([aktuelleStation.koordinate.lat, aktuelleStation.koordinate.lng], 18, { animate: true });
        }
    }

    // Aktuelle Position aktualisieren - GEÄNDERT: autoZentrieren deaktivieren nach erstem Aufruf
    function aktualisierePosition(position: GeolocationPosition) {
        console.log("Position aktualisiert", position.coords);
        aktuellePosition = {
            lat: position.coords.latitude,
            lng: position.coords.longitude
        };

        if (aktuellePosition && !spielBeendet) {
            entfernung = berechneEntfernung(aktuellePosition, aktuelleStation.koordinate);
            aktualisiereTippZustand();

            // Nur aktualisieren, wenn die Karte existiert
            if (map) {
                aktualisiereKarte();
            } else {
                console.log("Karte existiert noch nicht für Update");
            }

            // Nach erstem Update automatisches Zentrieren deaktivieren
            if (autoZentrieren && !standortAktualisieren) {
                setTimeout(() => {
                    autoZentrieren = false;
                }, 2000); // 2 Sekunden warten, dann deaktivieren
            }

            // Debug-Info aktualisieren
            if (testModus) {
                debugInfo = `
          Aktuelle Position: ${aktuellePosition.lat.toFixed(6)}, ${aktuellePosition.lng.toFixed(6)}
          Ziel: ${aktuelleStation.koordinate.lat.toFixed(6)}, ${aktuelleStation.koordinate.lng.toFixed(6)}
          Entfernung: ${entfernung.toFixed(2)} m
          Suchradius: ${suchRadius} m
          Station gefunden: ${positionGefunden ? 'Ja' : 'Nein'}
        `;
            }
        }
    }

    // Prüfen, ob der Spieler nahe genug an der Station ist
    function aktualisiereTippZustand() {
        if (entfernung <= suchRadius) {
            tippAnzeigen = true;
            positionGefunden = true; // Setze positionGefunden automatisch auf true
        } else {
            tippAnzeigen = false;
            positionGefunden = false;
        }
    }

    // TEST-MODUS: Position simulieren
    function aktualisiereTestPosition() {
        aktuellePosition = { ...testPosition };
        console.log("Testposition aktualisiert:", aktuellePosition);

        if (aktuellePosition && !spielBeendet) {
            entfernung = berechneEntfernung(aktuellePosition, aktuelleStation.koordinate);
            aktualisiereTippZustand();

            // Nur aktualisieren, wenn die Karte existiert
            if (map) {
                aktualisiereKarte();
            } else {
                console.log("Karte existiert noch nicht für Test-Update");
            }

            // Bewegung simulieren - langsam zur aktuellen Station bewegen
            const zielRichtung = Math.atan2(
                aktuelleStation.koordinate.lng - testPosition.lng,
                aktuelleStation.koordinate.lat - testPosition.lat
            );

            testPosition.lat += Math.cos(zielRichtung) * bewegungsVektor.lat;
            testPosition.lng += Math.sin(zielRichtung) * bewegungsVektor.lng;

            // Debug-Info aktualisieren
            debugInfo = `
        Test-Position: ${aktuellePosition.lat.toFixed(6)}, ${aktuellePosition.lng.toFixed(6)}
        Ziel: ${aktuelleStation.koordinate.lat.toFixed(6)}, ${aktuelleStation.koordinate.lng.toFixed(6)}
        Entfernung: ${entfernung.toFixed(2)} m
        Suchradius: ${suchRadius} m
        Station gefunden: ${positionGefunden ? 'Ja' : 'Nein'}
        Bewegungsvektor: ${bewegungsVektor.lat.toExponential(2)}, ${bewegungsVektor.lng.toExponential(2)}
        Simulationsgeschwindigkeit: ${simulationsGeschwindigkeit} ms
      `;
        }
    }

    // Antwort überprüfen
    function ueberpruefeAntwort() {
        const normalisierteAntwort = benutzerAntwort.trim().toLowerCase();
        const normalisierteLosung = aktuelleStation.loesung.trim().toLowerCase();

        if (normalisierteAntwort === normalisierteLosung) {
            geloesteListe = [...geloesteListe, aktuelleStation.id];
            erfolgNachricht = "Richtig! Das Rätsel wurde gelöst.";
            fehlerNachricht = "";

            // Speichere Fortschritt nach dem Lösen
            speichereFortschritt();

            // Wenn es weitere Stationen gibt, zur nächsten wechseln
            if (aktuelleStationIndex < stationen.length - 1) {
                setTimeout(() => {
                    aktuelleStationIndex++;
                    aktuelleStation = stationen[aktuelleStationIndex];
                    benutzerAntwort = "";
                    erfolgNachricht = "";
                    tippAnzeigen = false;
                    positionGefunden = false;
                    
                    // Beim Wechsel zur nächsten Station automatisches Zentrieren wieder aktivieren
                    autoZentrieren = true;
                    aktualisiereKarte();

                    // Speichere Fortschritt nach dem Wechsel zur nächsten Station
                    speichereFortschritt();
                }, 2000);
            } else {
                // Spiel beendet
                spielBeendet = true;
                speichereFortschritt();
            }
        } else {
            fehlerNachricht = "Das ist leider nicht richtig. Versuche es noch einmal!";
            erfolgNachricht = "";
        }
    }

    // Standortnutzung starten (wird durch Benutzerklick aufgerufen)
    function starteSpiel() {
        statusNachricht = "Starte Spiel...";
        autoZentrieren = true; // Beim Start automatisches Zentrieren aktivieren

        if (testModus) {
            // TEST-MODUS starten
            console.log("Starte im Test-Modus");
            spielGestartet = true;
            speichereFortschritt();

            // Verzögerung hinzufügen, um sicherzustellen, dass der DOM vollständig gerendert ist
            setTimeout(() => {
                initialisiereKarte(testPosition);
                aktualisiereTestPosition();

                // Intervall für simulierte Bewegung einrichten
                const testInterval = setInterval(() => {
                    aktualisiereTestPosition();
                }, simulationsGeschwindigkeit);

                // Aufräumen beim Unmount
                return () => {
                    clearInterval(testInterval);
                };
            }, 500);

        } else if (browser && navigator.geolocation) {
            statusNachricht = "Fordere Standorterlaubnis an...";
            console.log("Fordere Standorterlaubnis an...");

            // Einmalige Position abrufen (löst die Berechtigungsabfrage aus)
            navigator.geolocation.getCurrentPosition(
                (position) => {
                    console.log("Standort erhalten:", position.coords);
                    statusNachricht = "Standort gefunden!";
                    spielGestartet = true;
                    speichereFortschritt();

                    aktuellePosition = {
                        lat: position.coords.latitude,
                        lng: position.coords.longitude
                    };

                    // Kurze Verzögerung vor dem Initialisieren der Karte
                    setTimeout(() => {
                        initialisiereKarte(aktuellePosition);
                        aktualisierePosition(position);

                        // Kontinuierliche Positionsverfolgung starten
                        watchId = navigator.geolocation.watchPosition(
                            aktualisierePosition,
                            (error) => {
                                console.error("Fehler bei der Standortverfolgung:", error);
                            },
                            { enableHighAccuracy: true, maximumAge: 10000, timeout: 5000 }
                        );
                    }, 500);
                },
                (error) => {
                    console.error("Standortfehler:", error);
                    switch(error.code) {
                        case error.PERMISSION_DENIED:
                            positionError = "Standortzugriff verweigert. Du musst den Standortzugriff erlauben, um spielen zu können.";
                            break;
                        case error.POSITION_UNAVAILABLE:
                            positionError = "Standortinformationen sind nicht verfügbar.";
                            break;
                        case error.TIMEOUT:
                            positionError = "Zeitüberschreitung beim Abrufen des Standorts.";
                            break;
                        default:
                            positionError = "Unbekannter Fehler beim Ermitteln des Standorts.";
                            break;
                    }
                    statusNachricht = "Standortfehler aufgetreten";
                },
                { enableHighAccuracy: true }
            );
        } else {
            positionError = "Geolocation wird von deinem Browser nicht unterstützt.";
            statusNachricht = "Fehler: Browser unterstützt keine Standortabfrage";
        }
    }

    // Spiel fortsetzen mit gespeichertem Fortschritt
    function setzeSpielFort() {
        statusNachricht = "Setze Spiel fort...";
        autoZentrieren = true; // Beim Fortsetzen automatisches Zentrieren aktivieren

        if (testModus) {
            // TEST-MODUS fortsetzen
            console.log("Setze im Test-Modus fort");
            spielGestartet = true;
            speichereFortschritt();

            // Verzögerung hinzufügen, um sicherzustellen, dass der DOM vollständig gerendert ist
            setTimeout(() => {
                initialisiereKarte(testPosition);
                aktualisiereTestPosition();

                // Intervall für simulierte Bewegung einrichten
                const testInterval = setInterval(() => {
                    aktualisiereTestPosition();
                }, simulationsGeschwindigkeit);

                // Aufräumen beim Unmount
                return () => {
                    clearInterval(testInterval);
                };
            }, 500);
        } else if (browser && navigator.geolocation) {
            statusNachricht = "Fordere Standorterlaubnis an...";
            console.log("Fordere Standorterlaubnis an für Fortsetzung...");
            // Einmalige Position abrufen (löst die Berechtigungsabfrage aus)
            navigator.geolocation.getCurrentPosition(
                (position) => {
                    console.log("Standort erhalten für Fortsetzung:", position.coords);
                    statusNachricht = "Standort gefunden!";
                    spielGestartet = true;
                    speichereFortschritt();

                    aktuellePosition = {
                        lat: position.coords.latitude,
                        lng: position.coords.longitude
                    };

                    // Kurze Verzögerung vor dem Initialisieren der Karte
                    setTimeout(() => {
                        initialisiereKarte(aktuellePosition);
                        aktualisierePosition(position);

                        // Kontinuierliche Positionsverfolgung starten
                        watchId = navigator.geolocation.watchPosition(
                            aktualisierePosition,
                            (error) => {
                                console.error("Fehler bei der Standortverfolgung:", error);
                            },
                            { enableHighAccuracy: true, maximumAge: 10000, timeout: 5000 }
                        );
                    }, 500);
                },
                (error) => {
                    console.error("Standortfehler bei Fortsetzung:", error);
                    switch(error.code) {
                        case error.PERMISSION_DENIED:
                            positionError = "Standortzugriff verweigert. Du musst den Standortzugriff erlauben, um spielen zu können.";
                            break;
                        case error.POSITION_UNAVAILABLE:
                            positionError = "Standortinformationen sind nicht verfügbar.";
                            break;
                        case error.TIMEOUT:
                            positionError = "Zeitüberschreitung beim Abrufen des Standorts.";
                            break;
                        default:
                            positionError = "Unbekannter Fehler beim Ermitteln des Standorts.";
                            break;
                    }
                    statusNachricht = "Standortfehler aufgetreten";
                },
                { enableHighAccuracy: true }
            );
        } else {
            positionError = "Geolocation wird von deinem Browser nicht unterstützt.";
            statusNachricht = "Fehler: Browser unterstützt keine Standortabfrage";
        }
    }

    // Spiel neu starten
    function starteNeu() {
        // Stoppe aktuelle Positionsverfolgung falls vorhanden
        if (watchId !== null) {
            navigator.geolocation.clearWatch(watchId);
            watchId = null;
        }

        // Karte entfernen falls vorhanden
        if (map) {
            map.remove();
            map = null;
        }

        // Fortschritt zurücksetzen
        aktuelleStationIndex = 0;
        aktuelleStation = stationen[aktuelleStationIndex];
        geloesteListe = [];
        spielBeendet = false;
        spielGestartet = false;
        autoZentrieren = true; // Beim Neustart automatisches Zentrieren aktivieren
        
        // UI-Zustand zurücksetzen
        benutzerAntwort = "";
        fehlerNachricht = "";
        erfolgNachricht = "";
        tippAnzeigen = false;
        positionGefunden = false;
        positionError = "";
        aktuellePosition = null;
        entfernung = -1;

        // Lösche gespeicherten Fortschritt
        loescheFortschritt();

        // Kurze Verzögerung, dann Spiel neu starten
        setTimeout(() => {
            starteSpiel();
        }, 100);
    }

    // TEST-MODUS Einstellungen aktualisieren
    function aktualisiereTestEinstellungen() {
        suchRadius = Math.max(10, Math.min(1000, suchRadius)); // Zwischen 10 und 1000 Metern
        simulationsGeschwindigkeit = Math.max(500, Math.min(10000, simulationsGeschwindigkeit)); // Zwischen 0.5 und 10 Sekunden

        // Radiuskreis aktualisieren
        if (radiusKreis) {
            radiusKreis.setRadius(suchRadius);
        }

        // Debug-Info aktualisieren
        if (testModus && aktuellePosition) {
            debugInfo = `
        Test-Position: ${aktuellePosition.lat.toFixed(6)}, ${aktuellePosition.lng.toFixed(6)}
        Ziel: ${aktuelleStation.koordinate.lat.toFixed(6)}, ${aktuelleStation.koordinate.lng.toFixed(6)}
        Entfernung: ${entfernung.toFixed(2)} m
        Suchradius: ${suchRadius} m
        Station gefunden: ${positionGefunden ? 'Ja' : 'Nein'}
        Bewegungsvektor: ${bewegungsVektor.lat.toExponential(2)}, ${bewegungsVektor.lng.toExponential(2)}
        Simulationsgeschwindigkeit: ${simulationsGeschwindigkeit} ms
      `;
        }
    }

    // Teleportiere den Test-Spieler zur aktuellen Station (für einfaches Testen)
    function teleportiereZurStation() {
        if (testModus) {
            testPosition = {
                lat: aktuelleStation.koordinate.lat + 0.0001, // Ein bisschen daneben, damit es nicht sofort aktiviert wird
                lng: aktuelleStation.koordinate.lng + 0.0001
            };
            aktualisiereTestPosition();
        }
    }

    // Debug-Positionen direkt setzen
    function setzeTestPosition(lat: number, lng: number) {
        if (testModus) {
            testPosition = { lat, lng };
            aktualisiereTestPosition();
        }
    }

    onMount(() => {
        if (browser) {
            // URL-Parameter auslesen
            const urlParams = new URLSearchParams(window.location.search);
            testModus = urlParams.get('debug') === 'true';

            if (testModus) {
                statusNachricht = "Test-Modus über URL aktiviert. Standortdaten werden simuliert.";
                // Optional weitere Parameter auslesen
                const radius = urlParams.get('radius');
                if (radius) suchRadius = parseInt(radius);

                const speed = urlParams.get('speed');
                if (speed) simulationsGeschwindigkeit = parseInt(speed);

                const vectorLat = urlParams.get('vLat');
                if (vectorLat) bewegungsVektor.lat = parseFloat(vectorLat);

                const vectorLng = urlParams.get('vLng');
                if (vectorLng) bewegungsVektor.lng = parseFloat(vectorLng);
            }

            // Gespeicherten Fortschritt laden
            hatGespeichertenFortschritt = ladeFortschritt();

            if (hatGespeichertenFortschritt) {
                statusNachricht = "Gespeicherter Fortschritt gefunden! Du kannst weiterspielen.";
            }
        }

        // Aufräumen beim Unmount
        return () => {
            if (watchId !== null) {
                navigator.geolocation.clearWatch(watchId);
            }
            if (map) {
                map.remove();
                map = null;
            }
        };
    });
</script>

<div class="min-h-screen bg-black text-white flex flex-col">
    <!-- Haupt-App nach Uber-Design -->
    {#if spielGestartet}
        <!-- Statusleiste im Uber-Stil -->
        <div class="bg-black py-3 px-4 shadow-lg z-10 flex justify-between items-center">
            <div class="flex items-center">
                <h2 class="font-medium text-lg">{aktuelleStation.name}</h2>
                <div class="ml-2 px-2 py-0.5 bg-gray-800 rounded-full text-xs">
                    {aktuelleStationIndex + 1}/{stationen.length}
                </div>
            </div>

            {#if testModus}
                <div class="px-2 py-1 bg-yellow-500 rounded-md text-xs text-white font-medium">
                    TEST
                </div>
            {/if}
        </div>

        <!-- Karte im Vollbild-Modus (Uber-Stil) -->
        <div class="flex-grow relative">
            <div id="map" class="w-full h-full absolute inset-0">
                <!-- Leaflet-Karte wird hier eingefügt -->
            </div>

            <!-- NEU: Standort-Buttons im Uber-Stil (oben links) -->
            <div class="absolute top-3 left-3 flex flex-col space-y-2 z-10">
                <!-- Standort aktualisieren Button -->
                <button
                    on:click={aktualisiereStandortManuell}
                    class="bg-black hover:bg-gray-800 rounded-full shadow-lg p-3 transition-colors"
                    title="Standort aktualisieren"
                >
                    <svg xmlns="http://www.w3.org/2000/svg" class="h-5 w-5 text-white" fill="none" viewBox="0 0 24 24" stroke="currentColor">
                        <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M17.657 16.657L13.414 20.9a1.998 1.998 0 01-2.827 0l-4.244-4.243a8 8 0 1111.314 0z" />
                        <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M15 11a3 3 0 11-6 0 3 3 0 016 0z" />
                    </svg>
                </button>

                <!-- Auf Position zentrieren Button -->
                <button
                    on:click={zentriereAufPosition}
                    class="bg-black hover:bg-gray-800 rounded-full shadow-lg p-3 transition-colors"
                    title="Auf meine Position zentrieren"
                >
                    <svg xmlns="http://www.w3.org/2000/svg" class="h-5 w-5 text-blue-400" fill="none" viewBox="0 0 24 24" stroke="currentColor">
                        <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M12 19l9 2-9-18-9 18 9-2zm0 0v-8" />
                    </svg>
                </button>

                <!-- Auf Station zentrieren Button -->
                <button
                    on:click={zentriereAufStation}
                    class="bg-black hover:bg-gray-800 rounded-full shadow-lg p-3 transition-colors"
                    title="Auf Ziel zentrieren"
                >
                    <svg xmlns="http://www.w3.org/2000/svg" class="h-5 w-5 text-red-400" fill="none" viewBox="0 0 24 24" stroke="currentColor">
                        <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M7 12l3-3 3 3 4-4M8 21l4-4 4 4M3 4h18M4 4h16v12a1 1 0 01-1 1H5a1 1 0 01-1-1V4z" />
                    </svg>
                </button>
            </div>

            <!-- Entfernungsanzeige im Uber-Stil -->
            <div class="absolute top-3 right-3 bg-black rounded-full shadow-lg px-3 py-1 text-sm font-medium z-10">
                {#if entfernung >= 0}
                    {#if entfernung > 1000}
                        {(entfernung / 1000).toFixed(1)} km
                    {:else}
                        {Math.round(entfernung)} m
                    {/if}
                {:else}
                    Standort wird ermittelt...
                {/if}
            </div>

            <!-- Fehlermeldungen -->
            {#if positionError && !testModus}
                <div class="absolute top-16 left-3 right-3 bg-red-500 text-white p-3 rounded-lg z-10 shadow-lg">
                    <p>{positionError}</p>
                </div>
            {/if}
        </div>

        <!-- Bottom Sheet im Uber-Stil, erscheint wenn man nahe genug ist -->
        {#if tippAnzeigen}
            <div class="bg-white text-black rounded-t-xl shadow-lg z-20 transition-all duration-300 ease-in-out transform">
                <!-- Drag-Indicator im Uber-Stil -->
                <div class="flex justify-center py-2">
                    <div class="w-12 h-1 bg-gray-300 rounded-full"></div>
                </div>

                <div class="px-4 pb-8 pt-2">
                    <!-- Titel im Uber-Stil -->
                    <div class="flex justify-between items-center mb-4">
                        <h3 class="font-bold text-xl text-black">Ort erreicht!</h3>
                    </div>

                    <!-- Trennlinie im Uber-Stil -->
                    <div class="border-b border-gray-200 -mx-4 mb-4"></div>

                    <!-- Inhalt -->
                    <div>
                        <!-- Nur Tipp anzeigen -->
                        <p class="text-gray-800 text-base font-medium mb-2">Tipp:</p>
                        <p class="text-gray-700 text-base mb-6">{aktuelleStation.tipp}</p>

                        <!-- Hinweis, dass das Rätsel physisch vor Ort zu finden ist -->
                        <div class="bg-gray-100 rounded-lg p-3 mb-6">
                            <p class="text-gray-700 text-sm">Das Rätsel findest du am Ort. Gib hier die Lösung ein, wenn du sie gefunden hast.</p>
                        </div>

                        <!-- Input im Uber-Stil -->
                        <div class="space-y-4">
                            <div class="relative">
                                <input
                                        type="text"
                                        bind:value={benutzerAntwort}
                                        placeholder="Lösung eingeben"
                                        class="w-full p-3 bg-gray-100 rounded-lg text-black focus:outline-none focus:ring-2 focus:ring-black"
                                />
                            </div>

                            <!-- Uber-Style Button (schwarz mit voller Breite) -->
                            <button
                                    on:click={ueberpruefeAntwort}
                                    class="w-full bg-black hover:bg-gray-800 text-white py-3 px-4 rounded-lg font-medium transition-colors"
                            >
                                Lösung überprüfen
                            </button>
                        </div>

                        {#if fehlerNachricht}
                            <p class="mt-3 text-red-500">{fehlerNachricht}</p>
                        {/if}

                        {#if erfolgNachricht}
                            <p class="mt-3 text-green-500">{erfolgNachricht}</p>
                        {/if}
                    </div>
                </div>
            </div>
        {:else if !positionError}
            <!-- Bottom Mini-Card im Uber-Stil -->
            <div class="bg-white text-black rounded-t-xl shadow-lg p-4 z-20">
                <div class="flex items-center justify-between">
                    <div>
                        <p class="font-bold text-black">Auf dem Weg zum Ziel</p>
                        <p class="text-sm text-gray-500">Folge der Karte zur nächsten Station</p>
                    </div>
                    <div class="ml-4 flex-shrink-0">
                        {#if entfernung >= 0}
              <span class="font-bold text-lg">
                {#if entfernung > 1000}
                  {(entfernung / 1000).toFixed(1)} km
                {:else}
                  {Math.round(entfernung)} m
                {/if}
              </span>
                        {/if}
                    </div>
                </div>
            </div>
        {/if}

        <!-- Debug-Panel (nur im Test-Modus) - im Uber-Stil -->
        {#if testModus}
            <div class="absolute top-32 right-3 bg-black border border-gray-800 rounded-lg shadow-lg p-3 z-30 max-w-xs opacity-90 hover:opacity-100 transition-opacity">
                <button
                        class="absolute -top-2 -right-2 bg-gray-800 text-white w-5 h-5 rounded-full flex items-center justify-center text-xs font-bold"
                        on:click={() => document.querySelector('#debug-panel').classList.toggle('hidden')}
                >
                    ×
                </button>

                <div id="debug-panel">
                    <h3 class="text-xs font-bold mb-2 text-gray-400">DEBUG TOOLS</h3>

                    <div class="space-y-3 text-xs">
                        <div>
                            <label class="mb-1 block text-gray-400">Auto-Zoom: {autoZentrieren ? 'An' : 'Aus'}</label>
                            <button
                                    on:click={() => autoZentrieren = !autoZentrieren}
                                    class="w-full bg-gray-800 text-white py-1 px-2 rounded-md text-xs"
                            >
                                {autoZentrieren ? 'Deaktivieren' : 'Aktivieren'}
                            </button>
                        </div>

                        <div>
                            <label class="mb-1 block text-gray-400">Radius: {suchRadius}m</label>
                            <input
                                    type="range"
                                    min="10"
                                    max="1000"
                                    step="10"
                                    bind:value={suchRadius}
                                    on:change={aktualisiereTestEinstellungen}
                                    class="w-full h-1 bg-gray-700 rounded-lg appearance-none cursor-pointer"
                            />
                        </div>

                        <div class="flex space-x-1">
                            <button
                                    on:click={teleportiereZurStation}
                                    class="flex-1 bg-gray-800 text-white py-1 px-2 rounded-md text-xs"
                            >
                                Teleport
                            </button>
                            <button
                                    on:click={() => setzeTestPosition(aktuellePosition.lat + 0.001, aktuellePosition.lng)}
                                    class="flex-1 bg-gray-800 text-white py-1 px-2 rounded-md text-xs"
                            >
                                Nord
                            </button>
                            <button
                                    on:click={() => setzeTestPosition(aktuellePosition.lat, aktuellePosition.lng + 0.001)}
                                    class="flex-1 bg-gray-800 text-white py-1 px-2 rounded-md text-xs"
                            >
                                Ost
                            </button>
                        </div>
                    </div>
                </div>
            </div>
        {/if}

        <!-- Spiel beendet Anzeige im Uber-Stil -->
        {#if spielBeendet}
            <div class="fixed inset-0 bg-black bg-opacity-90 flex items-center justify-center z-50">
                <div class="bg-white rounded-xl p-8 max-w-sm mx-auto text-center text-black">
                    <div class="w-16 h-16 bg-black rounded-full flex items-center justify-center mx-auto mb-6">
                        <span class="text-2xl">🎉</span>
                    </div>
                    <h2 class="text-2xl font-bold mb-4">Glückwunsch!</h2>
                    <p class="text-gray-700 mb-6">Du hast alle Stationen gefunden und alle Rätsel gelöst!</p>
                    <p class="text-xl font-bold text-black">Alles Gute zum Geburtstag! 🎂</p>
                    <button
                            on:click={starteNeu}
                            class="w-full bg-black text-white py-3 px-4 rounded-lg font-medium mt-6"
                    >
                        Neustart
                    </button>
                </div>
            </div>
        {/if}

        <!-- Startbildschirm im Uber-Stil -->
    {:else}
        <div class="flex flex-col items-stretch justify-between min-h-screen bg-black p-4">
            <!-- Oberer Bereich mit Uber-typischem Header -->
            <div class="pt-12 pb-8 text-center">
                <h1 class="text-4xl font-bold mb-2">Geburtstags-Schnitzeljagd</h1>
                <p class="text-gray-400">Eine digitale Entdeckungstour</p>
            </div>

            <!-- Mittlerer Bereich mit Karten-Illustration -->
            <div class="flex-grow flex flex-col items-center justify-center py-8">
                <div class="w-64 h-64 bg-gray-800 rounded-full flex items-center justify-center mb-8">
                    <svg xmlns="http://www.w3.org/2000/svg" class="h-32 w-32 text-white" fill="none" viewBox="0 0 24 24" stroke="currentColor">
                        <path stroke-linecap="round" stroke-linejoin="round" stroke-width="1" d="M17.657 16.657L13.414 20.9a1.998 1.998 0 01-2.827 0l-4.244-4.243a8 8 0 1111.314 0z" />
                        <path stroke-linecap="round" stroke-linejoin="round" stroke-width="1" d="M15 11a3 3 0 11-6 0 3 3 0 016 0z" />
                    </svg>
                </div>

                <!-- How-to-Play Karten im Uber-Stil -->
                <div class="grid grid-cols-3 gap-4 w-full max-w-lg">
                    <div class="bg-gray-900 rounded-lg p-3 text-center">
                        <div class="text-2xl mb-2">📍</div>
                        <p class="text-xs text-gray-400">Folge der Karte zum Ziel</p>
                    </div>
                    <div class="bg-gray-900 rounded-lg p-3 text-center">
                        <div class="text-2xl mb-2">🔍</div>
                        <p class="text-xs text-gray-400">Finde das Rätsel vor Ort</p>
                    </div>
                    <div class="bg-gray-900 rounded-lg p-3 text-center">
                        <div class="text-2xl mb-2">✅</div>
                        <p class="text-xs text-gray-400">Gib die Lösung in der App ein</p>
                    </div>
                </div>
            </div>

            <!-- Unterer Bereich mit Button -->
            <div class="pb-12">
                {#if positionError}
                    <div class="bg-red-500 bg-opacity-80 p-3 rounded-lg mb-4">
                        <p>{positionError}</p>
                        <p class="text-sm mt-2">Aktiviere den Test-Modus mit ?debug=true in der URL</p>
                    </div>
                {/if}

                {#if testModus}
                    <div class="bg-yellow-500 bg-opacity-80 p-3 rounded-lg mb-4 text-sm">
                        <p>Test-Modus aktiv! (URL-Parameter: ?debug=true)</p>
                    </div>
                {/if}

                {#if hatGespeichertenFortschritt}
                    <div class="bg-blue-500 bg-opacity-80 p-3 rounded-lg mb-4 text-sm">
                        <p>Du hast einen gespeicherten Spielstand bei Station {aktuelleStationIndex + 1} von {stationen.length}.</p>
                    </div>
                {/if}

                <p class="text-gray-400 text-center mb-4">{statusNachricht}</p>

                {#if hatGespeichertenFortschritt}
                    <div class="grid grid-cols-2 gap-3">
                        <button
                                on:click={setzeSpielFort}
                                class="w-full bg-white text-black hover:bg-gray-100 py-4 rounded-lg font-bold text-lg transition-colors"
                        >
                            Fortsetzen
                        </button>
                        <button
                                on:click={starteNeu}
                                class="w-full bg-gray-700 text-white hover:bg-gray-600 py-4 rounded-lg font-bold text-lg transition-colors"
                        >
                            Neu starten
                        </button>
                    </div>
                {:else}
                    <button
                            on:click={starteSpiel}
                            class="w-full bg-white text-black hover:bg-gray-100 py-4 rounded-lg font-bold text-lg transition-colors"
                    >
                        Jetzt starten
                    </button>
                {/if}
            </div>
        </div>
    {/if}

    <!-- Füge Stile für Animationen und Kartenmarker hinzu -->
    <style>
        /* Map-Container explizit stylen */
        #map {
            width: 100%;
            height: 100%;
            z-index: 1;
            background-color: #1a1a1a;
        }

        /* Allgemeine Stile für Leaflet korrigieren */
        :global(.leaflet-container) {
            background: #1a1a1a !important;
            height: 100%;
            width: 100%;
        }

        /* Anpassungen für Leaflet-Marker */
        :global(.position-marker div) {
            box-shadow: 0 0 0 4px rgba(0, 0, 0, 0.2);
            transform-origin: center;
            animation: pulse 2s infinite;
        }

        @keyframes pulse {
            0% { transform: scale(1); opacity: 1; }
            50% { transform: scale(1.1); opacity: 0.9; }
            100% { transform: scale(1); opacity: 1; }
        }

        :global(.station-marker div) {
            transform-origin: center;
            animation: bounce 1s infinite alternate;
        }

        @keyframes bounce {
            from { transform: translateY(0); }
            to { transform: translateY(-6px); }
        }
    </style>
</div>
