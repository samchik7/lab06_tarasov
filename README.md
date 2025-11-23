using System;
using System.Collections.Generic;
using System.Linq;
using System.Net.Http;
using System.Text.Json;
using System.Threading.Tasks;

namespace WeatherDataCollector
{
    public struct Weather
    {
        public string Country { get; set; }
        public string Name { get; set; }
        public double Temperature { get; set; }
        public string Description { get; set; }
        public double Latitude { get; set; }
        public double Longitude { get; set; }
    }

    class Program
    {
        private const string API_KEY = "d2f6109b36ac9f44e14280bdfb4003d9";
        private const string BASE_URL = "https://api.openweathermap.org/data/2.5/weather";
        private static readonly HttpClient httpClient = new HttpClient();
        private static readonly Random random = new Random();

        static async Task Main(string[] args)
        {
            List<Weather> weatherData = await CollectWeatherData(50);

            Console.WriteLine("Собрано данных о погоде: " + weatherData.Count);
            Console.WriteLine();

            // 1. Страна с максимальной и минимальной температурой
            var maxTemp = weatherData.OrderByDescending(w => w.Temperature).First();
            var minTemp = weatherData.OrderBy(w => w.Temperature).First();

            Console.WriteLine($"Страна с максимальной температурой: {maxTemp.Country} ({maxTemp.Name}) - {maxTemp.Temperature:F2}°C");
            Console.WriteLine($"Страна с минимальной температурой: {minTemp.Country} ({minTemp.Name}) - {minTemp.Temperature:F2}°C");
            Console.WriteLine();

            // 2. Средняя температура в мире
            double averageTemp = weatherData.Average(w => w.Temperature);
            Console.WriteLine($"Средняя температура в мире: {averageTemp:F2}°C");
            Console.WriteLine();

            // 3. Количество стран в коллекции
            int uniqueCountries = weatherData.Select(w => w.Country).Distinct().Count();
            Console.WriteLine($"Количество уникальных стран в коллекции: {uniqueCountries}");
            Console.WriteLine();

            // 4. Первые найденные страны с определенными описаниями
            var clearSky = weatherData.FirstOrDefault(w => w.Description == "clear sky");
            var rain = weatherData.FirstOrDefault(w => w.Description == "rain");
            var fewClouds = weatherData.FirstOrDefault(w => w.Description == "few clouds");

            Console.WriteLine("Первые найденные местности с определенными описаниями:");
            if (!string.IsNullOrEmpty(clearSky.Country))
                Console.WriteLine($"Clear sky: {clearSky.Country} - {clearSky.Name}");
            if (!string.IsNullOrEmpty(rain.Country))
                Console.WriteLine($"Rain: {rain.Country} - {rain.Name}");
            if (!string.IsNullOrEmpty(fewClouds.Country))
                Console.WriteLine($"Few clouds: {fewClouds.Country} - {fewClouds.Name}");
        }

        static async Task<List<Weather>> CollectWeatherData(int count)
        {
            var weatherList = new List<Weather>();
            int attempts = 0;
            const int maxAttempts = 200; // Ограничим количество попыток

            while (weatherList.Count < count && attempts < maxAttempts)
            {
                attempts++;

                // Генерируем случайные координаты
                double lat = random.NextDouble() * 180 - 90; // -90 до 90
                double lon = random.NextDouble() * 360 - 180; // -180 до 180

                try
                {
                    var weather = await GetWeatherData(lat, lon);
                    if (!string.IsNullOrEmpty(weather.Country) && !string.IsNullOrEmpty(weather.Name))
                    {
                        weatherList.Add(weather);
                        Console.WriteLine($"Получены данные: {weather.Country} - {weather.Name} - {weather.Temperature:F2}°C");
                    }
                }
                catch (Exception ex)
                {
                    Console.WriteLine($"Ошибка при получении данных для координат ({lat:F2}, {lon:F2}): {ex.Message}");
                }

                // Небольшая задержка чтобы не превысить лимиты API
                await Task.Delay(100);
            }

            return weatherList;
        }

        static async Task<Weather> GetWeatherData(double lat, double lon)
        {
            string url = $"{BASE_URL}?lat={lat}&lon={lon}&appid={API_KEY}&units=metric";

            HttpResponseMessage response = await httpClient.GetAsync(url);
            response.EnsureSuccessStatusCode();

            string json = await response.Content.ReadAsStringAsync();

            using JsonDocument doc = JsonDocument.Parse(json);
            JsonElement root = doc.RootElement;

            
            if (!root.TryGetProperty("sys", out JsonElement sys) ||
                !sys.TryGetProperty("country", out JsonElement countryElem) ||
                !root.TryGetProperty("name", out JsonElement nameElem))
            {
                return new Weather(); // Возвращаем пустую структуру если нет страны или названия
            }

            string country = countryElem.GetString();
            string name = nameElem.GetString();

            // Если страна или название пустые, возвращаем пустую структуру
            if (string.IsNullOrEmpty(country) || string.IsNullOrEmpty(name))
            {
                return new Weather();
            }

            double temperature = root.GetProperty("main").GetProperty("temp").GetDouble();
            string description = root.GetProperty("weather")[0].GetProperty("description").GetString();

            return new Weather
            {
                Country = country,
                Name = name,
                Temperature = temperature,
                Description = description,
                Latitude = lat,
                Longitude = lon
            };
        }
    }
}
