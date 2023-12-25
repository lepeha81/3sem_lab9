# 3sem_lab9
## Homework
### Задание 1
Создайте многопоточное приложение для получения средних цен акций за год.
Используйте сайт https://finance.yahoo.com/ для получения дневных котировок списка акций из файла ticker.txt. Формат ссылки следующий:
https://query1.finance.yahoo.com/v7/finance/download/{Код_бумаги}?period1={Начальная_дата}&period2={Конечная_дата}&interval=1d&events=history&includeAdjustedClose=true
,где:
Код_бумаги – тикер из списка акций
Начальная_дата – метка времени начала запрашиваемого периода в UNIX формате (год назад).
Конечная_дата – метка времени конца запрашиваемого периода в UNIX формате (текущая дата).
Например, формат ссылки для AAPL:
https://query1.finance.yahoo.com/v7/finance/download/AAPL?period1=1629072000&period2=1660608000&interval=1d&events=history&includeAdjustedClose=true

По мере получения данных выполните запуск задачи(Task), которая будет считать среднюю цену акции за год (используйте среднее значение для каждого дня как (High+Low)/2. Сложите все полученные значения и поделите на число дней).
Результатом работы задачи будет являться среднее значение цены за год, которое необходимо вывести в файл в формате «Тикер:Цена». При этом обеспечьте потокобезопасный доступ к файлу между всеми задачами.


```
using System;

class lab091
{
    static readonly Mutex mutex = new Mutex();
    //Mutex используется для обеспечения взаимного исключения при работе с общими ресурсами, чтобы только один поток имел доступ к этим ресурсам в определенный момент времени.
    static async Task Main()
    {
        List<string> tickers = new List<string>();

        using (StreamReader reader = new StreamReader("ticker.txt"))
        //В данной строке создается экземпляр класса StreamReader для чтения из файла "ticker.txt"
        {
            string line;
            while ((line = reader.ReadLine()) != null)
            {//будет выполнять итерации до тех пор, пока ReadLine() не вернет null. Внутри цикла считываем каждую строку и добавляем ее в список tickers
                tickers.Add(line);
            }
        }

        using (HttpClient client = new HttpClient())
        //создается экземпляр класса HttpClient, который будет использоваться для отправки HTTP-запросов
        {
            List<Task> tasks = new List<Task>();

            foreach (string ticker in tickers)
            {
                tasks.Add(GetDataForTicker(client, ticker));
                //создается новая задача, вызывая метод GetDataForTicker() с передачей экземпляра HttpClient и текущего тикера. Задача добавляется в список tasks.
            }
            //ожидает завершения всех задач в списке tasks
            await Task.WhenAll(tasks);
        }
    }

    static async Task GetDataForTicker(HttpClient client, string ticker)
    {//выполняет запрос к API Yahoo Finance для получения данных по указанному тикеру. Метод принимает экземпляр HttpClient и тикер в качестве аргументов.
        try
        {//создаются переменные startDate и endDate, содержащие дату и время. startDate устанавливается на 1 год назад от текущей даты и времени, а endDate устанавливается на текущую дату и время
            DateTime startDate = DateTime.Now.AddYears(-1);
            DateTime endDate = DateTime.Now;
            //преобразуется каждая из дат startDate и endDate в соответствующее количество секунд, прошедших с начала Unix Time
            long startUnixTime = ((DateTimeOffset)startDate).ToUnixTimeSeconds();
            long endUnixTime = ((DateTimeOffset)endDate).ToUnixTimeSeconds();
            //создается строка url, содержащая адрес API Yahoo Finance с указанием тикера, начальной и конечной даты, интервала и других параметров.
            string url = $"https://query1.finance.yahoo.com/v7/finance/download/{ticker}?period1={startUnixTime}&period2={endUnixTime}&interval=1d&events=history&includeAdjustedClose=true";
            // отправляет GET-запрос к API Yahoo Finance с использованием созданного экземпляра HttpClient и указанного url
            HttpResponseMessage response = await client.GetAsync(url);
            //вызов убеждается, что статус код ответа response является успешным и не содержит ошибок. Если статус код не является успешным, будет сгенерировано исключение.
            response.EnsureSuccessStatusCode();
            string csvData = await response.Content.ReadAsStringAsync();
            //читает содержимое ответа как строку
            //В данной строке строка csvData разбивается на массив строк lines по символу новой строки '\n'
            string[] lines = csvData.Split('\n');
            double totalAveragePrice = 0.0;
            int totalRowCount = 0;

            for (int i = 1; i < lines.Length - 1; i++)
            {
                try
                {//разделяет текущую строку lines[i] на подстроки, используя символ "," в качестве разделителя, и сохраняет результат в массив values
                    string[] values = lines[i].Split(',');
                    //извлекается значение из массива values по индексу 2 и преобразуется в тип double
                    double high = Convert.ToDouble(values[2], new System.Globalization.CultureInfo("en-US"));
                    //использование английского формата чисел, чтобы распознавать десятичные разделители точки.
                    double low = Convert.ToDouble(values[3], new System.Globalization.CultureInfo("en-US"));
                    //средняя
                    double averagePrice = (high + low) / 2;

                    totalAveragePrice += averagePrice;
                    totalRowCount++;
                    //бщей средней цены.
                }
                catch (Exception ex)
                {
                    Console.WriteLine($"Ошибка при обработке строки для {ticker}: {ex.Message}");
                }

            }

            if (totalRowCount > 0)
            {
                double totalAverage = totalAveragePrice / totalRowCount;
                string result = $"{ticker}:{totalAverage}";
                //захват объекта mutex, чтобы предотвратить одновременный доступ к общему ресурсу другими потоками
                mutex.WaitOne();
                try
                {//результат записывается в файл "results.txt" с использованием метода File.AppendAllText. result представляет собой строку, содержащую результат средней цены для одного тикера
                    File.AppendAllText("results.txt", result + Environment.NewLine);
                }//добавления новой строки после каждого результата.
                finally
                {//освобождает объект mutex, позволяя другим потокам получить доступ к общему ресурсу
                    mutex.ReleaseMutex();
                }

                Console.WriteLine($"Средняя цена акции для {ticker} за год: {totalAverage}");
            }
            else
            {
                Console.WriteLine($"Для {ticker} нет данных за год.");
            }
        }
        catch (Exception ex)
        {
            Console.WriteLine($"Ошибка при обработке {ticker}: {ex.Message}");
        }
    }
}
```
### Задание 2
На основе Лабораторной работы №6 создайте графическое приложение, получающее текущую погоду в разных городах мира.
Используйте WinForms или WPF.
Загрузите список городов с координатами из файла city.txt.
Добавьте в интерфейс приложения 2 элемента – один для отображения списка городов, второй элемент – кнопку, по нажатию на которую происходит загрузка текущей погоды в выбранном по API из Лабораторной работы №6.
Используйте ключевые слова async/await для загрузки данных и обновления интерфейса приложения таким образом, чтобы графический интерфейс не блокировался на время загрузки.
Если разработка ведется на *NIX или MACOS, допускается использовать консольное приложение, тем не менее следует использовать ключевые слова async/await при реализации методов.
1
```
using System.Text.Json;
using System.Text.RegularExpressions;
using System.Windows.Forms;

namespace WeatherApp
{
    public partial class AppWeather : Form
    {
        private const string apiKey = "eab787e4b774ddb90e6b0a1939d09d53";
        private const string apiUrl = "https://api.openweathermap.org/data/2.5/weather";

        public AppWeather()
        {
            InitializeComponent();
        }

        private async void Form1_Load(object sender, EventArgs e)
        //обработчик события Load формы Form1. Он вызывает метод LoadCityDataAsync() асинхронно, чтобы загрузить данные о городах
        {
            await LoadCityDataAsync();
        }

        private async Task LoadCityDataAsync()
        {//метод, который загружает данные о городах из файла "city.txt" и добавляет их в элемент listBoxCities на форме.
         //Он использует ключевое слово await для ожидания выполнения асинхронной операции ReadCitiesFromFileAsync()
            try
            {
                var cities = await ReadCitiesFromFileAsync("city.txt");
                listBoxCities.Items.AddRange(cities.ToArray());
            }
            catch (Exception ex)
            {
                MessageBox.Show($"Error loading city data: {ex.Message}", "Error", MessageBoxButtons.OK, MessageBoxIcon.Error);
            }
        }

        private async void buttonLoadWeather_Click(object sender, EventArgs e)
        {//обработчик события Click кнопки buttonLoadWeather. Он отвечает за загрузку погодных данных для выбранного города.
         //Если не выбран город или данные о нем некорректные, выводится соответствующее сообщение об ошибке.
         //Затем происходит разделение строки с выбранным городом для получения координат, и если удастся извлечь 3
         //значения координат, происходит конвертация широты в тип double и дальнейшая обработка погодных данных.
            if (listBoxCities.SelectedItem == null)
            {
                MessageBox.Show("Please select a city", "Error", MessageBoxButtons.OK, MessageBoxIcon.Error);
                return;
            }

            var selectedCity = listBoxCities.SelectedItem.ToString();
            //получается выбранный элемент из элемента управления "listBoxCities" и конвертируется в строку
            var coordinates = selectedCity.Split(',');
            // разделяется строка "selectedCity" с использованием запятой (',') в качестве разделителя, затем полученный массив присваивается переменной "coordinates"

            if (coordinates.Length != 3)
            {//проверяется, имеет ли массив "coordinates" длину 3. Если нет, это означает, что данные о городе недействительны
             //и не могут быть использованы
                MessageBox.Show("Invalid city data", "Error", MessageBoxButtons.OK, MessageBoxIcon.Error);
                return;
            }

            double latitude = Convert.ToDouble(coordinates[1], new System.Globalization.CultureInfo("en-US"));
            //происходит попытка преобразования строки "coordinates[1]" в значение типа double, представляющее широту.
            //Используется объект класса CultureInfo для указания, что числовой формат должен интерпретироваться в соответствии
            //с правилами культуры "en-US"
            double longitude = Convert.ToDouble(coordinates[2], new System.Globalization.CultureInfo("en-US"));
            //преобразует строку "coordinates[2]" в значение типа double, представляющее долготу.

            var weather = await GetWeatherDataAsync(latitude, longitude);
            //вызывается асинхронный метод "GetWeatherDataAsync" для получения данных о погоде на основе переданных координат широты и долготы.
            //await указывает, что выполнение кода должно быть приостановлено до завершения этого метода.
            if (weather != null)
            {
                DisplayWeatherInfo(weather);
            }//вызывает метод "DisplayWeatherInfo", который отображает информацию о погоде на экране, используя полученные данные о погоде
            else
            {
                labelWeatherInfo.Text = "Failed to fetch weather data";
            }

            Refresh();
            // обновляет текущий интерфейс пользователя, например, чтобы отобразить изменения текста в элементе управления "labelWeatherInfo"
        }

        private async Task<List<string>> ReadCitiesFromFileAsync(string filePath)
        //принимает путь к файлу и возвращает список строк
        {
            try
            {
                List<string> cities = new List<string>();

                using (StreamReader reader = new StreamReader(filePath))
                //создается новый объект "StreamReader" для чтения файла, указанного в переменной "filePath"
                {
                    while (!reader.EndOfStream)
                    {//начинает цикл, который будет выполняться до тех пор, пока не будет достигнут конец файла.
                        string line = await reader.ReadLineAsync();
                        //асинхронно считывается очередная строка из файла и присваивается переменной "line"
                        string[] parts = line.Split('\t');
                        //строка "line" разделяется на части, используя символ табуляции ('\t') в качестве разделителя.
                        //Результат сохраняется в массиве "parts".

                        if (parts.Length == 2)
                        {//проверяет, содержит ли массив "parts" две части. Если условие выполняется, значит, строка была
                         //успешно разделена на две части - название города и координаты
                            string cityName = parts[0].Trim();
                            //сохраняет первую часть разделенной строки "parts" в переменную "cityName", удаляя при этом
                            //начальные и конечные пробелы с помощью метода "Trim()"
                            string coordinates = parts[1].Trim();

                            if (IsValidCoordinates(coordinates))
                            {//вызывается метод "IsValidCoordinates", который проверяет, являются ли координаты допустимыми.
                                cities.Add($"{cityName},{coordinates}");
                            }//создается строка, объединяющая название города и его координаты с помощью запятой, и эта строка
                             //добавляется в список "cities"
                            else
                            {
                                MessageBox.Show($"Invalid coordinates format in line: {line}", "Error", MessageBoxButtons.OK, MessageBoxIcon.Error);
                            }
                        }
                        else
                        {
                            MessageBox.Show($"Invalid line format: {line}", "Error", MessageBoxButtons.OK, MessageBoxIcon.Error);
                        }
                    }
                }

                return cities;
            }
            catch (Exception ex)
            {
                MessageBox.Show($"Error reading cities from file: {ex.Message}", "Error", MessageBoxButtons.OK, MessageBoxIcon.Error);
                return new List<string>();
            }
        }

        private bool IsValidCoordinates(string coordinates)
        //используется для проверки, являются ли координаты в переданной строке допустимыми
        {
            if (Regex.IsMatch(coordinates, @"^\s*-?\d+(\.\d+)?,\s*-?\d+(\.\d+)?\s*$"))
            //Регулярное выражение проверяет, что строка начинается с необязательных пробелов, может содержать отрицательное
            //значение числа, за которым может следовать десятичная дробная часть. Затем должна быть запятая, опять могут следовать
            //необязательные пробелы и такая же последовательность для второго числа. В конце строки также необязательны пробелы.
            {
                return true;
            }
            return false;
        }

        private async Task<Weather> GetWeatherDataAsync(double latitude, double longitude)
        //используется для получения данных о погоде с помощью API на основе переданных координат.
        {
            using (HttpClient client = new HttpClient())
            {//создает экземпляр класса HttpClient для отправки HTTP-запросов и автоматического освобождения ресурсов после
             //использования благодаря использованию using-блока.
                int maxAttempts = 10;
                //определяет максимальное количество попыток получить данные о погоде.
                int attempt = 0;

                while (attempt < maxAttempts)
                {
                    try
                    {
                        var response = await client.GetStringAsync($"{apiUrl}?lat={latitude}&lon={longitude}&appid={apiKey}&units=metric");
                        // используется метод GetStringAsync класса HttpClient для отправки GET-запроса по определенному URL-адресу,
                        // который включает координаты широты и долготы, API-ключ и запрашиваемую единицу измерения (в данном случае
                        // метрическая система). Полученный ответ сохраняется в переменной "response".
                        var weatherInfo = JsonSerializer.Deserialize<WeatherInfo>(response);
                        // используется метод Deserialize класса JsonSerializer для преобразования полученного ответа "response"
                        // в объект WeatherInfo. JsonSerializer - это класс, который позволяет сериализовать или десериализовать
                        // объекты JSON
                        if (weatherInfo != null && !string.IsNullOrEmpty(weatherInfo.sys.country))
                        {//проверяет, что объект "weatherInfo" не является пустым и поле "country" в свойстве "sys" объекта "weatherInfo" не является пустой 
                            string country = weatherInfo.sys.country;
                            //извлекается значение поля "country" в свойстве "sys" и присваивается переменной "country"
                            string name = weatherInfo.name;
                            //из объекта "weatherInfo" извлекается значение поля "temp" в свойстве "main" и присваивается переменной "temp"
                            double temp = weatherInfo.main.temp;
                            string description = weatherInfo.weather[0].description;
                            //из объекта "weatherInfo" извлекается значение поля "description" в первом элементе массива "weather"
                            //и присваивается переменной "description"
                            return new Weather
                            {
                                Country = country,
                                Name = name,
                                Temp = temp,
                                Description = description
                            };
                        }
                    }
                    catch (HttpRequestException ex)
                    {
                        MessageBox.Show($"HTTP request error: {ex.Message}", "Error", MessageBoxButtons.OK, MessageBoxIcon.Error);
                        attempt++;
                        //отображает модальное диалоговое окно с текстом "HTTP request error:" и сообщением об ошибке,
                        //которое содержится в свойстве Message объекта ex, то есть сообщении об ошибке, которую сгенерировал
                        //объект HttpRequestException
                    }
                    catch (JsonException ex)
                    {
                        MessageBox.Show($"JSON deserialization error: {ex.Message}", "Error", MessageBoxButtons.OK, MessageBoxIcon.Error);
                        attempt++;
                    }
                    catch (Exception ex)
                    {
                        MessageBox.Show($"An error occurred: {ex.Message}", "Error", MessageBoxButtons.OK, MessageBoxIcon.Error);
                        attempt++;
                    }
                }

                return null;
            }
        }

        private void DisplayWeatherInfo(Weather weather)
        {
            if (weather != null)
            {//объявляет переменную weatherInfoText и присваивает ей значение, составленное из разных свойств объекта weather.
             //Здесь используется строковая интерполяция для форматирования текста. Значения свойств Country, Name, Temp и
             //Description подставляются в соответствующие места в строке
                string weatherInfoText = $"Country: {weather.Country}\nCity: {weather.Name}\nTemperature: {weather.Temp}°C\nDescription: {weather.Description}";
                labelWeatherInfo.Text = weatherInfoText;
            }//устанавливает свойство Text объекта labelWeatherInfo равным значению переменной weatherInfoText.
             //Таким образом, текст на метке будет содержать информацию о погоде, если объект weather не равен null
            else
            {//устанавливает свойство Text объекта labelWeatherInfo на строку "Weather data not available.".
             //Это сообщение будет отображаться на метке, если объект weather равен null
                labelWeatherInfo.Text = "Weather data not available.";
            }
        }

    }

    public class Weather
    {
        public string Country { get; set; }
        public string Name { get; set; }
        public double Temp { get; set; }
        public string Description { get; set; }
    }

    public class WeatherInfo
    {
        public MainInfo main { get; set; }
        public WeatherDescription[] weather { get; set; }
        public string name { get; set; }
        public SysInfo sys { get; set; }
    }

    public class MainInfo
    {
        public double temp { get; set; }
    }

    public class WeatherDescription
    {
        public string description { get; set; }
    }

    public class SysInfo
    {
        public string country { get; set; }
    }

}

```
2

```
using System;
using System.Windows.Forms;

namespace WeatherApp
{
    static class lab092
    {
        [STAThread]
        //атрибут, который указывает, что основной поток приложения будет использовать единственный поток с точкой входа для
        //обработки сообщений Windows. 
        static void Main()
        {
            Application.EnableVisualStyles();
            //вызов метода EnableVisualStyles() позволяет приложению использовать внешний вид операционной системы для
            //элементов управления Windows Forms. Он активирует графические стили операционной системы для вашего приложения.
            Application.SetCompatibleTextRenderingDefault(false);
            //устанавливает значение по умолчанию для совместимого рендеринга текста в false. Это означает, что текст в
            //элементах управления Windows Forms будет рендериться с использованием совместимого рендерера с лучшей
            //производительностью.
            Application.Run(new AppWeather());
            //запускает приложение с помощью метода Run() из класса Application. В аргументе передается новый экземпляр
            //класса AppWeather, который является формой или окном приложения.
        }
    }
}

```
