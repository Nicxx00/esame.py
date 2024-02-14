import csv 
#importo il modulo csv, mi permette di leggere e scrivere sul file 
#se esiste e se riesce ad aprirlo

#definisco una nuova classe che eredita dalla classe base Exception
#la uso per gestire le eccezioni del codice
class ExamException(Exception):  
    pass

class CSVTimeSeriesFile:
#definisco il costruttore della classe
#che prende come argomento il nome con cui vado a richiamare il file csv
    def __init__(self, name):
        self.name = name

    def get_data(self):
#definisco il metodo get_data che legge i dati del file csv
#li restituisce come una lista di liste

#creo lista vuota che conterrà i dati letti dal file csv
        data = []
#variabile che tiene tracccia della data precedente 
#quindi per verificare che siano in ordine temporale
        prev_date = None
#utilizzo un try-except per gestire gli eventuali errori durante la lettura del file csv
#se il file non viene trovato o si verifica un altro errore, solleva un'eccezione
        try:
            my_file = open(self.name, 'r')
#csv.reader() è una funzione fornita dal modulo csv che viene utilizzata per leggere il file csv
#mi permette di iterare attraverso le righe del file csv
#restituisce ciascuna riga come lista di stringhe
#reader = [line.split(',') for line in my_file]
            reader = csv.reader(my_file)
#reader è il nome della variabile che scelgo per rappresentare l'oggetto restituito dalla funzione csv.reader()
            for row in reader:
                if len(row) != 2:
                    continue
                date = row[0]
                passengers = row[1]
                prev_date = date   # Aggiorniamo il timestamp precedente
#provo a convertire il numero di passeggeri in un intero
#altrimenti, se non è possibile, la riga viene ignorata e si passa al comando successivo
# Controllo per verificare se il timestamp è fuori ordine
                if prev_date is not None and date < prev_date:
                    raise ExamException('Timestamp fuori ordine o duplicato')

                try:
                    passengers = int(passengers)
                except ValueError:
                    continue
                data.append([date, passengers])

        except FileNotFoundError:
            raise ExamException('File non trovato')
        except Exception as e:
            raise ExamException(str(e))
        my_file.close()
        print(data)
        return data
#definisco una funzione che calcola gli incrementi nel numero medio di passeggeri tra due anni specifici
def compute_increments(time_series, first_year, last_year):

    try:
        first_year = int(first_year)
        last_year = int(last_year)
    except ValueError:
        raise ExamException('Anni forniti non validi')


#creo un set per inserire gli anni presenti nell'intervallo
#usiamo un set perchè non aggiunge i duplicati
    years_present = set()
    months_present = dict()

#il trattino mi permette di ingorare dei valori durante il ciclo
    for date, passengers in time_series:
        try:
            year = int(date.split('-')[0])
            year_months = int(date.split('-')[1])
            passengers_months = int(passengers)
        except(ValueError, IndexError):
            continue
        years_present.add(year)

        if year not in months_present:
            months_present[year] = [0, 0]
        months_present[year][0] = year_months
        months_present[year][1] += passengers_months
        print(years_present)
        print(months_present)

    if first_year not in years_present or last_year not in years_present:
        raise ExamException('Intervallo di tempo non presente nel file CSV')

    if len(years_present) == 2 and (months_present[first_year][1] == 0 or months_present[last_year][1] == 0):
        return []


# Trovo gli anni con misurazioni presenti
#devo trasformare il set in una lista affinchè io possa modificarla
#la funzione sorted mi permette di riordinare la lista in ordine crescente
    available_years = sorted(list(years_present))


# Verifica se ci sono misurazioni mancanti per uno o più anni
    for year in range(first_year, last_year +1):
        if months_present[year][1] == 0:
            del available_years[available_years.index(year)]


#creo un dizionario vuoto che conterrà gli incrementi calcolati
    increments = {}

#teniamo traccia dell'anno corrente durante l'iterazione
#inizalizzata a none perchè all'inizio non c'è nessun anno corrente
    current_year = None
#creo un dizionario vuoto in cui salverò il totale dei passeggeri 
#e il conteggio dei mesi per ogni anno nella serie temporale 
#la chiave è l'anno e il valore è una lista che 
#ha come primo elemento il totale dei passeggeri e come secondo il conteggio del mese attuale
    year_passenger_totals = {}
#creo un altro dizionario che mi conta quanti mesi ci sono per ciascun anno
    year_month_counts = {}

#itero attraverso gli elementi della serie temporale
#quando chiamo la funzione e passo la serie temporale come argomento, 
#esso contiene tutte le coppie di date e passeggeri estratte dal file csv
#attraverso l'iterazione della serie temporale posso accedere ai valori 
#che saranno le variabili locali in tale funzione
    for date, passengers in time_series:
#estraggo l'anno dalla data utilizzando il carattere '-' come separatore
        try:
            year = int(date.split('-')[0])
#indexerror quando tento di accedere a un indice che non esiste in una sequenza
        except (ValueError, IndexError):
            continue

#iniziallizzo l'anno corrente
        if current_year is None:
            current_year = year
#se l'anno non è presente come chiave nel dizionario la creo
        if year not in year_passenger_totals and year not in year_month_counts:
            year_passenger_totals[year] = [0, 0]  #[total_passengers, month_count]
            year_month_counts[year] = 0

        year_passenger_totals[year][0] += passengers
        year_passenger_totals[year][1] += 1
        year_month_counts[year] += 1
#se l'anno corrente è diverso dall'anno dall'elemento corrente 
#calcolo l'incremento e lo aggiungo al dizionario
        if year != current_year and year_month_counts[year] == months_present[year][0]:
#crea un nuova chiave (ovvero una stringa ottenuta concatenando i due valori 
#separati da trattino)
#assegna alla chiave il risultato della divisione 
#(il numero medio di passeggeri in quella speifica iterazione)
            prev_year = current_year
            current_year = year

            try:
                current_avg_passengers = year_passenger_totals[current_year][0] / year_passenger_totals[current_year][1]
                prev_avg_passengers = year_passenger_totals[prev_year][0] / year_passenger_totals[prev_year][1]


            except ZeroDivisionError:
                current_avg_passengers = 0
                prev_avg_passengers = 0


            increment = current_avg_passengers - prev_avg_passengers
            increments["{}-{}".format(prev_year, current_year)] = increment

    return increments

# Utilizzo
time_series_file = CSVTimeSeriesFile(name='data.csv')
time_series = time_series_file.get_data()
increments = compute_increments(time_series, '1949', '1951')
print(increments)

