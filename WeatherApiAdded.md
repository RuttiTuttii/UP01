Да, переделаю так:

Самый дешёвый и нормальный вариант для погоды — OpenWeather.
AI-поиск через OpenRouter/ProxyAPI оставил как переключаемый вариант, но он будет дороже и менее стабилен, потому что это LLM + веб-поиск, а не погодный API.

Выпадающий список будет такой:

OpenWeather API
ProxyAPI OpenRouter Search

В appsettings.json вставишь:

{
  "OpenWeather": {
    "ApiKey": "PASTE_OPENWEATHER_API_KEY_HERE"
  },
  "ProxyApi": {
    "ApiKey": "PASTE_PROXYAPI_KEY_HERE",
    "OpenRouterModel": "openrouter/free"
  }
}

Для OpenRouter я поставил openrouter/free, чтобы было максимально дешево. Поиск включается через openrouter:web_search, а количество результатов ограничено до 1, чтобы не жрало баланс.

Ниже файлы на замену/добавление.

⸻

1. Добавить файл

Models/WeatherApiProvider.cs

namespace Practice25_27.Models;
public enum WeatherApiProvider
{
    OpenWeather = 0,
    ProxyApiOpenRouter = 1
}

⸻

2. Заменить файл

Models/CitySearchResult.cs

namespace Practice25_27.Models;
public class CitySearchResult
{
    public string Name { get; set; } = string.Empty;
    public string? LocalName { get; set; }
    public string? State { get; set; }
    public string Country { get; set; } = string.Empty;
    public double Latitude { get; set; }
    public double Longitude { get; set; }
    public bool HasCoordinates => Math.Abs(Latitude) > 0.0001 || Math.Abs(Longitude) > 0.0001;
    public string DisplayName
    {
        get
        {
            var city = string.IsNullOrWhiteSpace(LocalName) ? Name : LocalName;
            var statePart = string.IsNullOrWhiteSpace(State) ? string.Empty : $", {State}";
            var countryPart = string.IsNullOrWhiteSpace(Country) ? string.Empty : $", {Country}";
            if (!HasCoordinates)
            {
                return $"{city}{statePart}{countryPart}";
            }
            return $"{city}{statePart}{countryPart} ({Latitude:0.####}, {Longitude:0.####})";
        }
    }
}

⸻

3. Заменить файл

Models/SavedCity.cs

namespace Practice25_27.Models;
public class SavedCity
{
    public string Name { get; set; } = string.Empty;
    public string Country { get; set; } = string.Empty;
    public double Latitude { get; set; }
    public double Longitude { get; set; }
    public WeatherApiProvider Provider { get; set; } = WeatherApiProvider.OpenWeather;
    public string QueryText { get; set; } = string.Empty;
}

⸻

4. Заменить файл

Models/WeatherCardModel.cs

using System;
using System.Threading.Tasks;
using Avalonia.Media.Imaging;
using Practice25_27.Helpers;
namespace Practice25_27.Models;
public class WeatherCardModel
{
    private Task<Bitmap?>? _imageTask;
    public string CityName { get; set; } = string.Empty;
    public string Country { get; set; } = string.Empty;
    public double Latitude { get; set; }
    public double Longitude { get; set; }
    public string Description { get; set; } = string.Empty;
    public double Temperature { get; set; }
    public double FeelsLike { get; set; }
    public int PressureHpa { get; set; }
    public int Humidity { get; set; }
    public double WindSpeed { get; set; }
    public int WindDegrees { get; set; }
    public int Cloudiness { get; set; }
    public double? RainOneHour { get; set; }
    public double? SnowOneHour { get; set; }
    public string IconCode { get; set; } = "01d";
    public DateTime UpdatedAt { get; set; } = DateTime.Now;
    public WeatherApiProvider Provider { get; set; } = WeatherApiProvider.OpenWeather;
    public string QueryText { get; set; } = string.Empty;
    public string ProviderText => Provider switch
    {
        WeatherApiProvider.OpenWeather => "Источник: OpenWeather API",
        WeatherApiProvider.ProxyApiOpenRouter => "Источник: ProxyAPI / OpenRouter AI Search",
        _ => "Источник: неизвестно"
    };
    public string TitleText => string.IsNullOrWhiteSpace(Country) ? CityName : $"{CityName}, {Country}";
    public string DescriptionText => string.IsNullOrWhiteSpace(Description) ? "Описание отсутствует" : ToCapitalized(Description);
    public string TemperatureText => $"Температура: {Temperature:0.#} °C";
    public string FeelsLikeText => $"Ощущается как: {FeelsLike:0.#} °C";
    public string PressureText => $"Давление: {PressureHpaToMmHg(PressureHpa)} мм рт. ст. ({PressureHpa} гПа)";
    public string HumidityText => $"Влажность: {Humidity}%";
    public string WindText => $"Ветер: {GetWindDirection(WindDegrees)}, {WindSpeed:0.#} м/с";
    public string CloudsText => $"Облачность: {Cloudiness}%";
    public string RainText => RainOneHour is null ? "Дождь: нет" : $"Дождь: {RainOneHour:0.#} мм/ч";
    public string SnowText => SnowOneHour is null ? "Снег: нет" : $"Снег: {SnowOneHour:0.#} мм/ч";
    public string UpdatedAtText => $"Обновлено: {UpdatedAt:HH:mm}";
    public string IconUrl => $"https://openweathermap.org/img/wn/{IconCode}@2x.png";
    public Task<Bitmap?> Image => _imageTask ??= ImageHelper.LoadFromWeb(IconUrl);
    public SavedCity ToSavedCity()
    {
        return new SavedCity
        {
            Name = CityName,
            Country = Country,
            Latitude = Latitude,
            Longitude = Longitude,
            Provider = Provider,
            QueryText = string.IsNullOrWhiteSpace(QueryText) ? CityName : QueryText
        };
    }
    private static int PressureHpaToMmHg(int pressureHpa)
    {
        return (int)Math.Round(pressureHpa * 0.750062);
    }
    private static string GetWindDirection(int degrees)
    {
        var directions = new[]
        {
            "северный",
            "северо-восточный",
            "восточный",
            "юго-восточный",
            "южный",
            "юго-западный",
            "западный",
            "северо-западный"
        };
        var index = (int)Math.Round(((degrees % 360) / 45.0)) % 8;
        return directions[index];
    }
    private static string ToCapitalized(string value)
    {
        if (string.IsNullOrWhiteSpace(value))
        {
            return string.Empty;
        }
        return char.ToUpper(value[0]) + value[1..];
    }
}

⸻

5. Заменить файл

Services/WeatherService.cs

using System;
using System.Collections.Generic;
using System.Globalization;
using System.IO;
using System.Linq;
using System.Net.Http;
using System.Net.Http.Headers;
using System.Text;
using System.Text.Json;
using System.Text.Json.Serialization;
using System.Threading.Tasks;
using Practice25_27.Models;
namespace Practice25_27.Services;
public class WeatherService
{
    private readonly HttpClient _httpClient = new();
    private readonly string _openWeatherApiKey;
    private readonly string _proxyApiKey;
    private readonly string _proxyOpenRouterModel;
    private static readonly JsonSerializerOptions JsonOptions = new()
    {
        PropertyNameCaseInsensitive = true
    };
    public WeatherService(
        string? openWeatherApiKey = null,
        string? proxyApiKey = null,
        string? proxyOpenRouterModel = null)
    {
        var settings = LoadSettings();
        _openWeatherApiKey = string.IsNullOrWhiteSpace(openWeatherApiKey)
            ? settings.OpenWeatherApiKey
            : openWeatherApiKey.Trim();
        _proxyApiKey = string.IsNullOrWhiteSpace(proxyApiKey)
            ? settings.ProxyApiKey
            : proxyApiKey.Trim();
        _proxyOpenRouterModel = string.IsNullOrWhiteSpace(proxyOpenRouterModel)
            ? settings.ProxyOpenRouterModel
            : proxyOpenRouterModel.Trim();
        if (string.IsNullOrWhiteSpace(_proxyOpenRouterModel))
        {
            _proxyOpenRouterModel = "openrouter/free";
        }
    }
    public Task<IReadOnlyList<CitySearchResult>> SearchCitiesAsync(
        string cityName,
        WeatherApiProvider provider)
    {
        return provider switch
        {
            WeatherApiProvider.OpenWeather => SearchCitiesByOpenWeatherAsync(cityName),
            WeatherApiProvider.ProxyApiOpenRouter => SearchCitiesByProxyOpenRouterAsync(cityName),
            _ => SearchCitiesByOpenWeatherAsync(cityName)
        };
    }
    public Task<WeatherCardModel> GetCurrentByCityNameAsync(
        string cityName,
        WeatherApiProvider provider)
    {
        return provider switch
        {
            WeatherApiProvider.OpenWeather => GetCurrentByOpenWeatherCityNameAsync(cityName),
            WeatherApiProvider.ProxyApiOpenRouter => GetCurrentByProxyOpenRouterAsync(cityName),
            _ => GetCurrentByOpenWeatherCityNameAsync(cityName)
        };
    }
    public Task<WeatherCardModel> GetCurrentByCoordinatesAsync(
        double latitude,
        double longitude,
        WeatherApiProvider provider = WeatherApiProvider.OpenWeather)
    {
        return provider switch
        {
            WeatherApiProvider.OpenWeather => GetCurrentByOpenWeatherCoordinatesAsync(latitude, longitude),
            _ => throw new InvalidOperationException("Для этого источника нужны не координаты, а название города.")
        };
    }
    public async Task<WeatherCardModel> GetCurrentBySearchResultAsync(
        CitySearchResult city,
        WeatherApiProvider provider)
    {
        if (provider == WeatherApiProvider.OpenWeather)
        {
            return await GetCurrentByOpenWeatherCoordinatesAsync(city.Latitude, city.Longitude);
        }
        return await GetCurrentByProxyOpenRouterAsync(city.Name);
    }
    public async Task<WeatherCardModel> GetCurrentBySavedCityAsync(SavedCity city)
    {
        if (city.Provider == WeatherApiProvider.ProxyApiOpenRouter)
        {
            var query = string.IsNullOrWhiteSpace(city.QueryText)
                ? city.Name
                : city.QueryText;
            return await GetCurrentByProxyOpenRouterAsync(query);
        }
        return await GetCurrentByOpenWeatherCoordinatesAsync(city.Latitude, city.Longitude);
    }
    private async Task<IReadOnlyList<CitySearchResult>> SearchCitiesByOpenWeatherAsync(string cityName)
    {
        EnsureOpenWeatherApiKey();
        if (string.IsNullOrWhiteSpace(cityName))
        {
            return Array.Empty<CitySearchResult>();
        }
        var query = Uri.EscapeDataString(cityName.Trim());
        var url = $"https://api.openweathermap.org/geo/1.0/direct?q={query}&limit=10&appid={_openWeatherApiKey}";
        var cities = await GetFromJsonAsync<List<GeocodingItem>>(url) ?? new List<GeocodingItem>();
        return cities.Select(city => new CitySearchResult
            {
                Name = city.Name ?? string.Empty,
                LocalName = city.LocalNames?.TryGetValue("ru", out var russianName) == true ? russianName : city.Name,
                State = city.State,
                Country = city.Country ?? string.Empty,
                Latitude = city.Latitude,
                Longitude = city.Longitude
            })
            .ToList();
    }
    private Task<IReadOnlyList<CitySearchResult>> SearchCitiesByProxyOpenRouterAsync(string cityName)
    {
        if (string.IsNullOrWhiteSpace(cityName))
        {
            return Task.FromResult<IReadOnlyList<CitySearchResult>>(Array.Empty<CitySearchResult>());
        }
        IReadOnlyList<CitySearchResult> result = new List<CitySearchResult>
        {
            new()
            {
                Name = cityName.Trim(),
                LocalName = cityName.Trim(),
                State = "поиск через AI",
                Country = "ProxyAPI / OpenRouter",
                Latitude = 0,
                Longitude = 0
            }
        };
        return Task.FromResult(result);
    }
    private async Task<WeatherCardModel> GetCurrentByOpenWeatherCityNameAsync(string cityName)
    {
        EnsureOpenWeatherApiKey();
        if (string.IsNullOrWhiteSpace(cityName))
        {
            throw new ArgumentException("Введите название города.", nameof(cityName));
        }
        var query = Uri.EscapeDataString(cityName.Trim());
        var url = $"https://api.openweathermap.org/data/2.5/weather?q={query}&appid={_openWeatherApiKey}&units=metric&lang=ru";
        var response = await GetFromJsonAsync<CurrentWeatherResponse>(url);
        return MapOpenWeatherToWeatherCard(response);
    }
    private async Task<WeatherCardModel> GetCurrentByOpenWeatherCoordinatesAsync(double latitude, double longitude)
    {
        EnsureOpenWeatherApiKey();
        var lat = latitude.ToString(CultureInfo.InvariantCulture);
        var lon = longitude.ToString(CultureInfo.InvariantCulture);
        var url = $"https://api.openweathermap.org/data/2.5/weather?lat={lat}&lon={lon}&appid={_openWeatherApiKey}&units=metric&lang=ru";
        var response = await GetFromJsonAsync<CurrentWeatherResponse>(url);
        return MapOpenWeatherToWeatherCard(response);
    }
    private async Task<WeatherCardModel> GetCurrentByProxyOpenRouterAsync(string cityName)
    {
        EnsureProxyApiKey();
        if (string.IsNullOrWhiteSpace(cityName))
        {
            throw new ArgumentException("Введите название города.", nameof(cityName));
        }
        var requestBody = new
        {
            model = _proxyOpenRouterModel,
            temperature = 0,
            max_tokens = 350,
            messages = new object[]
            {
                new
                {
                    role = "system",
                    content = """
                    Ты работаешь как погодный API. 
                    Нужно найти актуальную текущую погоду в городе через веб-поиск.
                    Ответ должен быть только JSON без markdown, без пояснений, без ```.
                    Формат строго такой:
                    {
                      "cityName": "Название города",
                      "country": "Страна или код страны",
                      "description": "краткое описание погоды на русском",
                      "temperature": 12.3,
                      "feelsLike": 11.2,
                      "pressureHpa": 1013,
                      "humidity": 70,
                      "windSpeed": 4.5,
                      "windDegrees": 270,
                      "cloudiness": 80,
                      "rainOneHour": null,
                      "snowOneHour": null,
                      "iconCode": "04d"
                    }
                    Единицы:
                    - temperature и feelsLike: градусы Цельсия
                    - pressureHpa: гектопаскали
                    - humidity и cloudiness: проценты
                    - windSpeed: м/с
                    - rainOneHour и snowOneHour: мм/ч или null
                    iconCode выбери приблизительно по погоде:
                    01d ясно, 02d малооблачно, 03d облачно, 04d пасмурно,
                    09d дождь, 10d дождь с прояснениями, 11d гроза,
                    13d снег, 50d туман.
                    """
                },
                new
                {
                    role = "user",
                    content = $"Найди текущую погоду в городе: {cityName.Trim()}. Используй веб-поиск и верни только JSON."
                }
            },
            tools = new object[]
            {
                new
                {
                    type = "openrouter:web_search",
                    parameters = new
                    {
                        engine = "exa",
                        max_results = 1,
                        max_total_results = 1,
                        search_context_size = "low"
                    }
                }
            }
        };
        var json = JsonSerializer.Serialize(requestBody);
        using var request = new HttpRequestMessage(HttpMethod.Post, "https://api.proxyapi.ru/openrouter/v1/chat/completions");
        request.Headers.Authorization = new AuthenticationHeaderValue("Bearer", _proxyApiKey);
        request.Content = new StringContent(json, Encoding.UTF8, "application/json");
        using var response = await _httpClient.SendAsync(request);
        var responseJson = await response.Content.ReadAsStringAsync();
        if (!response.IsSuccessStatusCode)
        {
            var message = TryReadApiMessage(responseJson);
            throw new InvalidOperationException(string.IsNullOrWhiteSpace(message)
                ? $"ProxyAPI вернул ошибку: {(int)response.StatusCode} {response.ReasonPhrase}"
                : $"ProxyAPI вернул ошибку: {message}");
        }
        var content = ReadOpenRouterMessageContent(responseJson);
        if (string.IsNullOrWhiteSpace(content))
        {
            throw new InvalidOperationException("ProxyAPI вернул пустой ответ.");
        }
        var weatherJson = ExtractJsonObject(content);
        var proxyWeather = JsonSerializer.Deserialize<ProxyWeatherResponse>(weatherJson, JsonOptions);
        if (proxyWeather is null)
        {
            throw new InvalidOperationException("Не удалось прочитать JSON с погодой от ProxyAPI.");
        }
        return MapProxyWeatherToWeatherCard(proxyWeather, cityName);
    }
    private async Task<T?> GetFromJsonAsync<T>(string url)
    {
        using var response = await _httpClient.GetAsync(url);
        var json = await response.Content.ReadAsStringAsync();
        if (!response.IsSuccessStatusCode)
        {
            var message = TryReadApiMessage(json);
            throw new InvalidOperationException(string.IsNullOrWhiteSpace(message)
                ? $"API вернул ошибку: {(int)response.StatusCode} {response.ReasonPhrase}"
                : $"API вернул ошибку: {message}");
        }
        return JsonSerializer.Deserialize<T>(json, JsonOptions);
    }
    private static WeatherCardModel MapOpenWeatherToWeatherCard(CurrentWeatherResponse? response)
    {
        if (response is null)
        {
            throw new InvalidOperationException("Не удалось прочитать ответ API.");
        }
        var weather = response.Weather?.FirstOrDefault();
        return new WeatherCardModel
        {
            Provider = WeatherApiProvider.OpenWeather,
            QueryText = response.Name ?? string.Empty,
            CityName = response.Name ?? "Без названия",
            Country = response.Sys?.Country ?? string.Empty,
            Latitude = response.Coord?.Latitude ?? 0,
            Longitude = response.Coord?.Longitude ?? 0,
            Description = weather?.Description ?? string.Empty,
            IconCode = string.IsNullOrWhiteSpace(weather?.Icon) ? "01d" : weather.Icon,
            Temperature = response.Main?.Temperature ?? 0,
            FeelsLike = response.Main?.FeelsLike ?? 0,
            PressureHpa = response.Main?.Pressure ?? 0,
            Humidity = response.Main?.Humidity ?? 0,
            WindSpeed = response.Wind?.Speed ?? 0,
            WindDegrees = response.Wind?.Degrees ?? 0,
            Cloudiness = response.Clouds?.All ?? 0,
            RainOneHour = response.Rain?.OneHour,
            SnowOneHour = response.Snow?.OneHour,
            UpdatedAt = DateTime.Now
        };
    }
    private static WeatherCardModel MapProxyWeatherToWeatherCard(
        ProxyWeatherResponse response,
        string fallbackCityName)
    {
        return new WeatherCardModel
        {
            Provider = WeatherApiProvider.ProxyApiOpenRouter,
            QueryText = fallbackCityName,
            CityName = string.IsNullOrWhiteSpace(response.CityName) ? fallbackCityName : response.CityName,
            Country = response.Country ?? string.Empty,
            Latitude = 0,
            Longitude = 0,
            Description = response.Description ?? "Погода получена через AI-поиск",
            IconCode = string.IsNullOrWhiteSpace(response.IconCode) ? "02d" : response.IconCode,
            Temperature = response.Temperature ?? 0,
            FeelsLike = response.FeelsLike ?? response.Temperature ?? 0,
            PressureHpa = response.PressureHpa ?? 0,
            Humidity = response.Humidity ?? 0,
            WindSpeed = response.WindSpeed ?? 0,
            WindDegrees = response.WindDegrees ?? 0,
            Cloudiness = response.Cloudiness ?? 0,
            RainOneHour = response.RainOneHour,
            SnowOneHour = response.SnowOneHour,
            UpdatedAt = DateTime.Now
        };
    }
    private static string ReadOpenRouterMessageContent(string responseJson)
    {
        try
        {
            using var document = JsonDocument.Parse(responseJson);
            var root = document.RootElement;
            if (root.TryGetProperty("error", out var errorElement))
            {
                if (errorElement.TryGetProperty("message", out var errorMessage))
                {
                    return errorMessage.GetString() ?? string.Empty;
                }
            }
            var contentElement = root
                .GetProperty("choices")[0]
                .GetProperty("message")
                .GetProperty("content");
            return JsonElementToString(contentElement);
        }
        catch
        {
            return string.Empty;
        }
    }
    private static string JsonElementToString(JsonElement element)
    {
        if (element.ValueKind == JsonValueKind.String)
        {
            return element.GetString() ?? string.Empty;
        }
        if (element.ValueKind == JsonValueKind.Array)
        {
            var parts = new List<string>();
            foreach (var item in element.EnumerateArray())
            {
                if (item.ValueKind == JsonValueKind.String)
                {
                    parts.Add(item.GetString() ?? string.Empty);
                }
                else if (item.ValueKind == JsonValueKind.Object)
                {
                    if (item.TryGetProperty("text", out var textElement))
                    {
                        parts.Add(JsonElementToString(textElement));
                    }
                    else if (item.TryGetProperty("content", out var contentElement))
                    {
                        parts.Add(JsonElementToString(contentElement));
                    }
                }
            }
            return string.Join(Environment.NewLine, parts);
        }
        return element.ToString();
    }
    private static string ExtractJsonObject(string text)
    {
        var start = text.IndexOf('{');
        var end = text.LastIndexOf('}');
        if (start < 0 || end <= start)
        {
            throw new InvalidOperationException("Ответ AI не содержит JSON-объект.");
        }
        return text[start..(end + 1)];
    }
    private static WeatherServiceSettings LoadSettings()
    {
        var result = new WeatherServiceSettings();
        var possiblePaths = new[]
        {
            Path.Combine(AppContext.BaseDirectory, "appsettings.json"),
            Path.Combine(Directory.GetCurrentDirectory(), "appsettings.json")
        };
        foreach (var path in possiblePaths)
        {
            if (!File.Exists(path))
            {
                continue;
            }
            using var stream = File.OpenRead(path);
            using var document = JsonDocument.Parse(stream);
            var root = document.RootElement;
            if (root.TryGetProperty("ApiKey", out var legacyApiKeyElement))
            {
                var apiKey = legacyApiKeyElement.GetString();
                if (!IsPlaceholderOrEmpty(apiKey, "PASTE_OPENWEATHER_API_KEY_HERE"))
                {
                    result.OpenWeatherApiKey = apiKey!.Trim();
                }
            }
            if (root.TryGetProperty("OpenWeather", out var openWeatherElement)
                && openWeatherElement.TryGetProperty("ApiKey", out var openWeatherApiKeyElement))
            {
                var apiKey = openWeatherApiKeyElement.GetString();
                if (!IsPlaceholderOrEmpty(apiKey, "PASTE_OPENWEATHER_API_KEY_HERE"))
                {
                    result.OpenWeatherApiKey = apiKey!.Trim();
                }
            }
            if (root.TryGetProperty("ProxyApi", out var proxyApiElement))
            {
                if (proxyApiElement.TryGetProperty("ApiKey", out var proxyApiKeyElement))
                {
                    var apiKey = proxyApiKeyElement.GetString();
                    if (!IsPlaceholderOrEmpty(apiKey, "PASTE_PROXYAPI_KEY_HERE"))
                    {
                        result.ProxyApiKey = apiKey!.Trim();
                    }
                }
                if (proxyApiElement.TryGetProperty("OpenRouterModel", out var modelElement))
                {
                    var model = modelElement.GetString();
                    if (!string.IsNullOrWhiteSpace(model))
                    {
                        result.ProxyOpenRouterModel = model.Trim();
                    }
                }
            }
        }
        if (string.IsNullOrWhiteSpace(result.ProxyOpenRouterModel))
        {
            result.ProxyOpenRouterModel = "openrouter/free";
        }
        return result;
    }
    private void EnsureOpenWeatherApiKey()
    {
        if (IsPlaceholderOrEmpty(_openWeatherApiKey, "PASTE_OPENWEATHER_API_KEY_HERE"))
        {
            throw new InvalidOperationException("Укажите OpenWeather API-ключ в файле appsettings.json.");
        }
    }
    private void EnsureProxyApiKey()
    {
        if (IsPlaceholderOrEmpty(_proxyApiKey, "PASTE_PROXYAPI_KEY_HERE"))
        {
            throw new InvalidOperationException("Укажите ProxyAPI-ключ в файле appsettings.json.");
        }
    }
    private static bool IsPlaceholderOrEmpty(string? value, string placeholder)
    {
        return string.IsNullOrWhiteSpace(value)
               || value.Contains(placeholder, StringComparison.OrdinalIgnoreCase)
               || value.Contains("ВАШ_КЛЮЧ", StringComparison.OrdinalIgnoreCase);
    }
    private static string? TryReadApiMessage(string json)
    {
        try
        {
            using var document = JsonDocument.Parse(json);
            if (document.RootElement.TryGetProperty("message", out var messageElement))
            {
                return messageElement.GetString();
            }
            if (document.RootElement.TryGetProperty("error", out var errorElement))
            {
                if (errorElement.ValueKind == JsonValueKind.String)
                {
                    return errorElement.GetString();
                }
                if (errorElement.TryGetProperty("message", out var errorMessageElement))
                {
                    return errorMessageElement.GetString();
                }
            }
        }
        catch
        {
            return null;
        }
        return null;
    }
    private class WeatherServiceSettings
    {
        public string OpenWeatherApiKey { get; set; } = string.Empty;
        public string ProxyApiKey { get; set; } = string.Empty;
        public string ProxyOpenRouterModel { get; set; } = "openrouter/free";
    }
    private class GeocodingItem
    {
        [JsonPropertyName("name")]
        public string? Name { get; set; }
        [JsonPropertyName("local_names")]
        public Dictionary<string, string>? LocalNames { get; set; }
        [JsonPropertyName("lat")]
        public double Latitude { get; set; }
        [JsonPropertyName("lon")]
        public double Longitude { get; set; }
        [JsonPropertyName("country")]
        public string? Country { get; set; }
        [JsonPropertyName("state")]
        public string? State { get; set; }
    }
    private class CurrentWeatherResponse
    {
        [JsonPropertyName("coord")]
        public CoordInfo? Coord { get; set; }
        [JsonPropertyName("weather")]
        public List<WeatherInfo>? Weather { get; set; }
        [JsonPropertyName("main")]
        public MainInfo? Main { get; set; }
        [JsonPropertyName("wind")]
        public WindInfo? Wind { get; set; }
        [JsonPropertyName("clouds")]
        public CloudsInfo? Clouds { get; set; }
        [JsonPropertyName("rain")]
        public PrecipitationInfo? Rain { get; set; }
        [JsonPropertyName("snow")]
        public PrecipitationInfo? Snow { get; set; }
        [JsonPropertyName("sys")]
        public SysInfo? Sys { get; set; }
        [JsonPropertyName("name")]
        public string? Name { get; set; }
    }
    private class CoordInfo
    {
        [JsonPropertyName("lat")]
        public double Latitude { get; set; }
        [JsonPropertyName("lon")]
        public double Longitude { get; set; }
    }
    private class WeatherInfo
    {
        [JsonPropertyName("description")]
        public string? Description { get; set; }
        [JsonPropertyName("icon")]
        public string? Icon { get; set; }
    }
    private class MainInfo
    {
        [JsonPropertyName("temp")]
        public double Temperature { get; set; }
        [JsonPropertyName("feels_like")]
        public double FeelsLike { get; set; }
        [JsonPropertyName("pressure")]
        public int Pressure { get; set; }
        [JsonPropertyName("humidity")]
        public int Humidity { get; set; }
    }
    private class WindInfo
    {
        [JsonPropertyName("speed")]
        public double Speed { get; set; }
        [JsonPropertyName("deg")]
        public int Degrees { get; set; }
    }
    private class CloudsInfo
    {
        [JsonPropertyName("all")]
        public int All { get; set; }
    }
    private class PrecipitationInfo
    {
        [JsonPropertyName("1h")]
        public double? OneHour { get; set; }
    }
    private class SysInfo
    {
        [JsonPropertyName("country")]
        public string? Country { get; set; }
    }
    private class ProxyWeatherResponse
    {
        public string? CityName { get; set; }
        public string? Country { get; set; }
        public string? Description { get; set; }
        public double? Temperature { get; set; }
        public double? FeelsLike { get; set; }
        public int? PressureHpa { get; set; }
        public int? Humidity { get; set; }
        public double? WindSpeed { get; set; }
        public int? WindDegrees { get; set; }
        public int? Cloudiness { get; set; }
        public double? RainOneHour { get; set; }
        public double? SnowOneHour { get; set; }
        public string? IconCode { get; set; }
    }
}

⸻

6. Заменить файл

ViewModels/WeatherViewModel.cs

using System;
using System.Collections.Generic;
using System.Collections.ObjectModel;
using System.Linq;
using System.Threading.Tasks;
using CommunityToolkit.Mvvm.Input;
using Practice25_27.Helpers;
using Practice25_27.Models;
using Practice25_27.Services;
namespace Practice25_27.ViewModels;
public class WeatherViewModel : ViewModelBase
{
    private const string SavedCitiesKey = "weather_cards_cities";
    private readonly WeatherService _weatherService;
    private string _searchText = string.Empty;
    private CitySearchResult? _selectedCity;
    private WeatherProviderOption _selectedProvider;
    private string _statusMessage = string.Empty;
    private bool _isBusy;
    private TaskNotifier? _initializationTask;
    public WeatherViewModel(WeatherService weatherService)
    {
        _weatherService = weatherService;
        WeatherProviders = new ObservableCollection<WeatherProviderOption>
        {
            new()
            {
                Provider = WeatherApiProvider.OpenWeather,
                DisplayName = "OpenWeather API",
                Description = "Точный погодный API, самый нормальный вариант для этой лабы"
            },
            new()
            {
                Provider = WeatherApiProvider.ProxyApiOpenRouter,
                DisplayName = "ProxyAPI OpenRouter Search",
                Description = "AI + веб-поиск, дешевый режим через openrouter/free"
            }
        };
        _selectedProvider = WeatherProviders[0];
        SearchCommand = new AsyncRelayCommand(SearchCitiesAsync, CanSearchCities);
        AddSelectedCityCommand = new AsyncRelayCommand(AddSelectedCityAsync, CanAddSelectedCity);
        RefreshCardCommand = new AsyncRelayCommand<WeatherCardModel?>(RefreshCardAsync);
        DeleteCardCommand = new AsyncRelayCommand<WeatherCardModel?>(DeleteCardAsync);
        InitializationTask = InitializeAsync();
    }
    public ObservableCollection<WeatherProviderOption> WeatherProviders { get; }
    public ObservableCollection<CitySearchResult> Cities { get; } = new();
    public ObservableCollection<WeatherCardModel> Forecasts { get; } = new();
    public Task? InitializationTask
    {
        get => _initializationTask;
        private set => SetPropertyAndNotifyOnCompletion(ref _initializationTask, value);
    }
    public WeatherProviderOption SelectedProvider
    {
        get => _selectedProvider;
        set
        {
            if (SetProperty(ref _selectedProvider, value))
            {
                Cities.Clear();
                SelectedCity = null;
                StatusMessage = $"Выбран источник: {SelectedProvider.DisplayName}.";
                SearchCommand.NotifyCanExecuteChanged();
                AddSelectedCityCommand.NotifyCanExecuteChanged();
            }
        }
    }
    public string SearchText
    {
        get => _searchText;
        set
        {
            if (SetProperty(ref _searchText, value))
            {
                SearchCommand.NotifyCanExecuteChanged();
            }
        }
    }
    public CitySearchResult? SelectedCity
    {
        get => _selectedCity;
        set
        {
            if (SetProperty(ref _selectedCity, value))
            {
                AddSelectedCityCommand.NotifyCanExecuteChanged();
            }
        }
    }
    public string StatusMessage
    {
        get => _statusMessage;
        set => SetProperty(ref _statusMessage, value);
    }
    public bool IsBusy
    {
        get => _isBusy;
        set
        {
            if (SetProperty(ref _isBusy, value))
            {
                SearchCommand.NotifyCanExecuteChanged();
                AddSelectedCityCommand.NotifyCanExecuteChanged();
            }
        }
    }
    public IAsyncRelayCommand SearchCommand { get; }
    public IAsyncRelayCommand AddSelectedCityCommand { get; }
    public IAsyncRelayCommand<WeatherCardModel?> RefreshCardCommand { get; }
    public IAsyncRelayCommand<WeatherCardModel?> DeleteCardCommand { get; }
    private async Task InitializeAsync()
    {
        var savedCities = await Preferences.Load(SavedCitiesKey, new List<SavedCity>());
        if (savedCities.Count == 0)
        {
            StatusMessage = "Введите город, выберите источник и нажмите «Поиск».";
            return;
        }
        IsBusy = true;
        try
        {
            foreach (var city in savedCities)
            {
                try
                {
                    var weather = await _weatherService.GetCurrentBySavedCityAsync(city);
                    Forecasts.Add(weather);
                }
                catch
                {
                    Forecasts.Add(new WeatherCardModel
                    {
                        CityName = city.Name,
                        Country = city.Country,
                        Latitude = city.Latitude,
                        Longitude = city.Longitude,
                        Provider = city.Provider,
                        QueryText = string.IsNullOrWhiteSpace(city.QueryText) ? city.Name : city.QueryText,
                        Description = "Не удалось обновить данные",
                        UpdatedAt = DateTime.Now
                    });
                }
            }
            StatusMessage = "Сохраненные города загружены.";
        }
        finally
        {
            IsBusy = false;
        }
    }
    private bool CanSearchCities()
    {
        return !IsBusy && !string.IsNullOrWhiteSpace(SearchText);
    }
    private bool CanAddSelectedCity()
    {
        return !IsBusy && SelectedCity is not null;
    }
    private async Task SearchCitiesAsync()
    {
        IsBusy = true;
        StatusMessage = "Ищу города...";
        Cities.Clear();
        SelectedCity = null;
        try
        {
            var cities = await _weatherService.SearchCitiesAsync(SearchText, SelectedProvider.Provider);
            foreach (var city in cities)
            {
                Cities.Add(city);
            }
            if (Cities.Count == 0)
            {
                StatusMessage = "Города не найдены. Проверьте название.";
            }
            else if (SelectedProvider.Provider == WeatherApiProvider.ProxyApiOpenRouter)
            {
                SelectedCity = Cities[0];
                StatusMessage = "Для AI-поиска создан один вариант. Нажмите «Добавить карточку».";
            }
            else
            {
                StatusMessage = "Выберите город из списка.";
            }
        }
        catch (Exception ex)
        {
            StatusMessage = ex.Message;
        }
        finally
        {
            IsBusy = false;
        }
    }
    private async Task AddSelectedCityAsync()
    {
        if (SelectedCity is null)
        {
            return;
        }
        if (Forecasts.Any(item => IsSameCard(item, SelectedCity, SelectedProvider.Provider)))
        {
            StatusMessage = "Этот город уже есть в списке.";
            return;
        }
        IsBusy = true;
        StatusMessage = "Загружаю погоду...";
        try
        {
            var weather = await _weatherService.GetCurrentBySearchResultAsync(
                SelectedCity,
                SelectedProvider.Provider);
            Forecasts.Add(weather);
            await SaveForecastCitiesAsync();
            StatusMessage = SelectedProvider.Provider == WeatherApiProvider.ProxyApiOpenRouter
                ? "Карточка добавлена через ProxyAPI / OpenRouter Search."
                : "Карточка добавлена через OpenWeather API.";
        }
        catch (Exception ex)
        {
            StatusMessage = ex.Message;
        }
        finally
        {
            IsBusy = false;
        }
    }
    private async Task RefreshCardAsync(WeatherCardModel? card)
    {
        if (card is null)
        {
            return;
        }
        var index = Forecasts.IndexOf(card);
        if (index < 0)
        {
            return;
        }
        try
        {
            WeatherCardModel updatedCard;
            if (card.Provider == WeatherApiProvider.ProxyApiOpenRouter)
            {
                var query = string.IsNullOrWhiteSpace(card.QueryText)
                    ? card.CityName
                    : card.QueryText;
                updatedCard = await _weatherService.GetCurrentByCityNameAsync(
                    query,
                    WeatherApiProvider.ProxyApiOpenRouter);
            }
            else
            {
                updatedCard = await _weatherService.GetCurrentByCoordinatesAsync(
                    card.Latitude,
                    card.Longitude,
                    WeatherApiProvider.OpenWeather);
            }
            Forecasts[index] = updatedCard;
            await SaveForecastCitiesAsync();
            StatusMessage = $"Карточка «{updatedCard.TitleText}» обновлена.";
        }
        catch (Exception ex)
        {
            StatusMessage = ex.Message;
        }
    }
    private async Task DeleteCardAsync(WeatherCardModel? card)
    {
        if (card is null)
        {
            return;
        }
        Forecasts.Remove(card);
        await SaveForecastCitiesAsync();
        StatusMessage = "Карточка удалена из списка.";
    }
    private async Task SaveForecastCitiesAsync()
    {
        var savedCities = Forecasts
            .Select(item => item.ToSavedCity())
            .ToList();
        await Preferences.Save(SavedCitiesKey, savedCities);
    }
    private static bool IsSameCard(
        WeatherCardModel card,
        CitySearchResult city,
        WeatherApiProvider provider)
    {
        if (card.Provider != provider)
        {
            return false;
        }
        if (provider == WeatherApiProvider.OpenWeather)
        {
            return Math.Abs(card.Latitude - city.Latitude) < 0.0001
                   && Math.Abs(card.Longitude - city.Longitude) < 0.0001;
        }
        return string.Equals(
            NormalizeCityName(card.QueryText),
            NormalizeCityName(city.Name),
            StringComparison.OrdinalIgnoreCase);
    }
    private static string NormalizeCityName(string value)
    {
        return value.Trim().ToLowerInvariant();
    }
}
public class WeatherProviderOption
{
    public WeatherApiProvider Provider { get; init; }
    public string DisplayName { get; init; } = string.Empty;
    public string Description { get; init; } = string.Empty;
}

⸻

7. Заменить файл

Views/WeatherView.axaml

<UserControl xmlns="https://github.com/avaloniaui"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
             x:Class="Practice25_27.Views.WeatherView">
  <Grid RowDefinitions="Auto,Auto,*" Margin="20" RowSpacing="14">
    <StackPanel Grid.Row="0" Spacing="6">
      <TextBlock Text="Получение данных по сети" FontSize="22" FontWeight="SemiBold" />
      <TextBlock Text="Поиск города, получение текущей погоды, карточки прогноза и сохранение списка городов через Preferences." TextWrapping="Wrap" />
    </StackPanel>
    <Border Grid.Row="1" Padding="14" CornerRadius="10" BorderBrush="#33000000" BorderThickness="1">
      <Grid ColumnDefinitions="*,Auto" RowDefinitions="Auto,Auto,Auto,Auto" ColumnSpacing="10" RowSpacing="10">
        <ComboBox Grid.Row="0"
                  Grid.ColumnSpan="2"
                  MinHeight="38"
                  PlaceholderText="Выберите источник погоды"
                  ItemsSource="{Binding WeatherProviders}"
                  SelectedItem="{Binding SelectedProvider}">
          <ComboBox.ItemTemplate>
            <DataTemplate>
              <StackPanel Spacing="2">
                <TextBlock Text="{Binding DisplayName}" FontWeight="SemiBold" />
                <TextBlock Text="{Binding Description}" FontSize="12" Opacity="0.75" TextWrapping="Wrap" />
              </StackPanel>
            </DataTemplate>
          </ComboBox.ItemTemplate>
        </ComboBox>
        <TextBox Grid.Row="1"
                 Grid.Column="0"
                 Watermark="Введите город, например Amsterdam"
                 Text="{Binding SearchText}" />
        <Button Grid.Row="1"
                Grid.Column="1"
                Content="Поиск"
                Command="{Binding SearchCommand}" />
        <ComboBox Grid.Row="2"
                  Grid.Column="0"
                  MinHeight="36"
                  PlaceholderText="Выберите город из найденных"
                  ItemsSource="{Binding Cities}"
                  SelectedItem="{Binding SelectedCity}">
          <ComboBox.ItemTemplate>
            <DataTemplate>
              <TextBlock Text="{Binding DisplayName}" TextWrapping="Wrap" />
            </DataTemplate>
          </ComboBox.ItemTemplate>
        </ComboBox>
        <Button Grid.Row="2"
                Grid.Column="1"
                Content="Добавить карточку"
                Command="{Binding AddSelectedCityCommand}" />
        <TextBlock Grid.Row="3"
                   Grid.ColumnSpan="2"
                   Text="{Binding StatusMessage}"
                   TextWrapping="Wrap" />
      </Grid>
    </Border>
    <ScrollViewer Grid.Row="2">
      <ItemsControl ItemsSource="{Binding Forecasts}">
        <ItemsControl.ItemsPanel>
          <ItemsPanelTemplate>
            <WrapPanel />
          </ItemsPanelTemplate>
        </ItemsControl.ItemsPanel>
        <ItemsControl.ItemTemplate>
          <DataTemplate>
            <Border Width="320"
                    MinHeight="410"
                    Margin="0,0,14,14"
                    Padding="14"
                    CornerRadius="12"
                    BorderBrush="#33000000"
                    BorderThickness="1">
              <Grid RowDefinitions="Auto,Auto,Auto,Auto,*,Auto" RowSpacing="8">
                <Grid Grid.Row="0" ColumnDefinitions="*,Auto">
                  <StackPanel Grid.Column="0" Spacing="2">
                    <TextBlock Text="{Binding TitleText}" FontSize="20" FontWeight="SemiBold" TextWrapping="Wrap" />
                    <TextBlock Text="{Binding DescriptionText}" TextWrapping="Wrap" />
                    <TextBlock Text="{Binding ProviderText}" FontSize="12" Opacity="0.7" TextWrapping="Wrap" />
                  </StackPanel>
                  <Image Grid.Column="1" Width="70" Height="70" Source="{Binding Image^}" />
                </Grid>
                <StackPanel Grid.Row="1" Spacing="4">
                  <TextBlock Text="{Binding TemperatureText}" />
                  <TextBlock Text="{Binding FeelsLikeText}" />
                  <TextBlock Text="{Binding PressureText}" />
                  <TextBlock Text="{Binding HumidityText}" />
                  <TextBlock Text="{Binding WindText}" />
                </StackPanel>
                <StackPanel Grid.Row="2" Spacing="4">
                  <TextBlock Text="{Binding CloudsText}" />
                  <TextBlock Text="{Binding RainText}" />
                  <TextBlock Text="{Binding SnowText}" />
                </StackPanel>
                <TextBlock Grid.Row="3" Text="{Binding UpdatedAtText}" Opacity="0.7" />
                <StackPanel Grid.Row="5" Orientation="Horizontal" Spacing="8">
                  <Button Content="Обновить"
                          Command="{Binding DataContext.RefreshCardCommand, RelativeSource={RelativeSource AncestorType=UserControl}}"
                          CommandParameter="{Binding}" />
                  <Button Content="Удалить"
                          Command="{Binding DataContext.DeleteCardCommand, RelativeSource={RelativeSource AncestorType=UserControl}}"
                          CommandParameter="{Binding}" />
                </StackPanel>
              </Grid>
            </Border>
          </DataTemplate>
        </ItemsControl.ItemTemplate>
      </ItemsControl>
    </ScrollViewer>
  </Grid>
</UserControl>

⸻

8. Заменить файл

appsettings.json

{
  "OpenWeather": {
    "ApiKey": "PASTE_OPENWEATHER_API_KEY_HERE"
  },
  "ProxyApi": {
    "ApiKey": "PASTE_PROXYAPI_KEY_HERE",
    "OpenRouterModel": "openrouter/free"
  }
}

⸻

ProxyAPI требует отправлять запросы на https://api.proxyapi.ru, а ключ передается через Authorization: Bearer <КЛЮЧ>. Для OpenRouter через ProxyAPI базовый путь — https://api.proxyapi.ru/openrouter/v1, а формат совместим с OpenAI Chat Completions.  ￼

OpenRouter сейчас рекомендует для веб-поиска серверный инструмент openrouter:web_search; я ограничил max_results и max_total_results до 1, чтобы запросы были дешевле.  ￼