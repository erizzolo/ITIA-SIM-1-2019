# Soluzione esempio seconda prova
## Variante con specializzazione di Noleggio
In questo caso la specializzazione può essere basata sulla avvenuta o meno conclusione del noleggio:
* Entità Noleggio(generica) con attributo tempo inizio, che partecipa alle associazioni Interesse (1:1) con Bicicletta (0:1), Responsabilità (1:1) con Utente (0:N) e Prelievo (1:1) con Stazione (0:N)
* Entità NoleggioInCorso senza altri attributi
* Entità NoleggioConcluso con attributo ulteriore tempo fine, che partecipa all'associazione Riconsegna (1:1) con Stazione (0:N)
Si osservi però che all'associazione Prelievo l'entità Bicicletta partecipa con cardinalità (0:1) se il noleggio è in corso, (0:N) se il noleggio è concluso.
### Progettazione logica
L'osservazione precedente ci permette di tradurre la specializzazione Noleggio in modo *particolare*, e cioè inserendo i dati relativi all'eventuale noleggio in corso nella tabella Bicicletta, e tutti i dati in una tabella NoleggioConcluso separata. Utilizzando per brevità la versione di Stazione con chiave surrogata, si arriva al seguente schema logico:
* Stazione(**id**, indirizzo!, [totali], [liberi], [occupati])
* Utente (**id**, cognome, nome, telefono, email!, username!, password, smartcard!, card type, card number, card expiry, report interval, report type, last report)
* Bicicletta(**id**, *stazione*, *utente*?, tempoPrelievo?)
* NoleggioConcluso(***bicicletta***, ***utente***, *stazionePrelievo*, **tempoPrelievo**, *stazioneRiconsegna*, tempoRiconsegna, costo)

Si noti che l'attributo stazione in Bicicletta assumerà ora due significati (eventualmente distinguibili con campi calcolati e/o view):
* stazione attuale se la bibicletta non è in uso: utente e tempoPrelievo NULL
* stazione di prelievo se la bibicletta è in uso: utente e tempoPrelievo NOT NULL
### Progettazione fisica
Con questa possibilità cambiano ovviamente le operazioni per prelievo e riconsegna, ed i relativi trigger.  
* [ ] E' possibile far corrispondere ciascuna operazione ad un semplice update di Bicicletta e gestire in un unico trigger i diversi casi: prelievo | riconsegna.  

Prelievo della bicicletta con codice @BICI da parte dell'utente con codice @USER:
````sql
UPDATE Bicicletta
  SET utente = @USER, tempoPrelievo = NOW()
  WHERE id = @BICI;
````
Riconsegna della bicicletta con codice @BICI alla stazione codice @STATION [da parte dell'utente con codice @USER]:
````sql
UPDATE Bicicletta
  SET stazione = @STATION, utente = NULL, dataOraPrelievo = NULL
  WHERE id = @BICI;
````
L'aggiornamento sarà legato ad un unico trigger che provvede a mantenere coerenti i dati del db:
````sql

````

Si inserisce il codice per la creazione del database anche se non richiesto dalla traccia:
````sql
DROP DATABASE IF EXISTS DBBikeRental;
CREATE DATABASE  IF NOT EXISTS DBBikeRental;
USE DBBikeRental
CREATE TABLE Utente (
  id INT AUTO_INCREMENT COMMENT "Unique auto generated user id",
  nome VARCHAR(50) NOT NULL COMMENT "Nome utente",
  cognome VARCHAR(50) NOT NULL COMMENT "Cognome utente",
  email VARCHAR(50) NOT NULL COMMENT "Indirizzo e-mail utente",
  telefono VARCHAR(50) NOT NULL COMMENT "Telefono utente",
  username VARCHAR(50) NOT NULL COMMENT "Nome login utente",
  password VARCHAR(128) NOT NULL COMMENT "Password hash SHA-2 512 bits",
  -- e.g. SHA2('myPassword',512)
  smartcard VARCHAR(10) NOT NULL COMMENT "Smart card code",
  cardType VARCHAR(10) NOT NULL COMMENT "Credit card type",
  cardNumber VARCHAR(20) NOT NULL COMMENT "Credit card number",
  cardExpiry DATE NOT NULL COMMENT "Credit card expiry date",
  reportInterval INT NOT NULL COMMENT "Report interval in days?!",
  reportType INT NOT NULL COMMENT "Report type ...",
  lastReport DATE NULL DEFAULT NULL COMMENT "Last report date",
  UNIQUE Smartcard (smartcard), -- no duplicates
  UNIQUE LoginIdentifier (username), -- no duplicates
  UNIQUE MailIdentifier (email), -- no duplicates
  PRIMARY KEY(id)
);
CREATE TABLE Stazione (
  id INT AUTO_INCREMENT COMMENT "Unique station id",
  indirizzo VARCHAR(100) NOT NULL COMMENT "Indirizzo stazione",
  liberi INT NOT NULL DEFAULT 0 COMMENT "Slot liberi",
  occupati INT NOT NULL DEFAULT 50 COMMENT "Slot occupati",
  totali INT NOT NULL DEFAULT 50 COMMENT "Slot totali",
  PRIMARY KEY(id),
    CONSTRAINT TertiumNonDatur CHECK(liberi + occupati = totali),
  UNIQUE Placement (indirizzo) -- no two stations at the same address
);
CREATE TABLE Bicicletta (
  id INT AUTO_INCREMENT COMMENT "Unique bike id",
  stazione INT NULL COMMENT "Stazione attuale",
  utente INT NULL COMMENT "User if currently rented",
  tempoPrelievo TIMESTAMP NULL COMMENT "Start of current rental if any",
  PRIMARY KEY(id),
  CONSTRAINT RentingUser FOREIGN KEY (utente) REFERENCES Utente(id)
    ON UPDATE CASCADE ON DELETE RESTRICT,
  CONSTRAINT Position FOREIGN KEY (stazione) REFERENCES Stazione(id)
    ON UPDATE CASCADE ON DELETE RESTRICT
);
CREATE TABLE NoleggioConcluso (
  bicicletta INT NOT NULL COMMENT "Bike id",
  utente INT NOT NULL COMMENT "User id",
  stazPrelievo INT NOT NULL COMMENT "Stazione prelievo",
  tempoPrelievo TIMESTAMP NOT NULL COMMENT "Istante prelievo",
  stazRiconsegna INT NOT NULL COMMENT "Stazione riconsegna",
  tempoRiconsegna TIMESTAMP NOT NULL COMMENT "Istante riconsegna",
  costo DECIMAL(6,2) NOT NULL COMMENT "Costo noleggio, dopo riconsegna",
  PRIMARY KEY(bicicletta, utente, tempoPrelievo),
  CONSTRAINT Utilizzo FOREIGN KEY (bicicletta) REFERENCES Bicicletta(id)
    ON UPDATE CASCADE ON DELETE RESTRICT,
  CONSTRAINT Prelievo FOREIGN KEY (stazPrelievo) REFERENCES Stazione(id)
    ON UPDATE CASCADE ON DELETE RESTRICT,
  CONSTRAINT Riconsegna FOREIGN KEY (stazPrelievo) REFERENCES Stazione(id)
    ON UPDATE CASCADE ON DELETE RESTRICT,
  CONSTRAINT Responsabile FOREIGN KEY (ytente) REFERENCES Utente(id)
    ON UPDATE CASCADE ON DELETE RESTRICT
);
DELIMITER //
CREATE TRIGGER Rental AFTER UPDATE ON Bicicletta FOR EACH ROW
  BEGIN
    IF IS NULL OLD.utente
      -- prelievo
        UPDATE Stazione SET liberi = liberi + 1, occupati = occupati - 1
          WHERE id = NEW.stazione;
    ELSE
      -- riconsegna
        UPDATE Stazione SET liberi = liberi - 1, occupati = occupati + 1
          WHERE id = NEW.stazione;
        -- calcolo costo
        SET @COSTO = 5; -- forfait 5.00 €
        INSERT INTO NoleggioConcluso
          VALUES(NEW.id, OLD.utente, OLD.stazione, OLD.tempoPrelievo, NEW.stazione, NOW(), @COSTO);
    END
  END;
//
DELIMITER ;
````
## Progettazione pagine web
idem
### a) mappa delle stazioni
idem
### b) biciclette in uso
idem
### Codifica pagine web
idem ma la query diventa più semplice:
Per semplicità si sceglie di codificare la pagina relativa alla richiesta b).
````sql
SELECT *
  FROM Bicicletta b
    JOIN Utente u ON (u.id = b.utente)
    JOIN Stazione s ON (s.id = b.stazione);
````
Sono escluse dal JOIN (INNER) le biciclette non in uso.

Per il resto non cambia quasi nulla.
