Este codigo utiliza algunas libreriras y conceptos de python para funcionar.
Es un programa que se comunica con la api de spotify para acceder a las top 10 canciones del artista que le pongas.

La idea general de este programa es hacer una funcion que retorne data de un artista que vamos a necesitar. Su nombre, y su id. Lo vamos a necesitar porque a la hora de pedir top 10 canciones, con otra funcion, a esa otra funcion habra que pasarle el id.
Ten en cuenta que esa funcion tb retorna un json lo cual habra qe transformar a diccionario y formatear, en una variable aparte, al final.

Usa 5 librerias:

dotenv: se usa para mantener privada y segura data sensible.
os: se lo utiliza para interactuar con variables del entorno.
base64: provee encodificacion y decodificacion en base 64, que por alguna razon, la api de spotify pide que asi pasemos las keys.
requests: Hace solicitudes Http (GET y POST) para interactuar con una api.
json: Convierte un archivo json a un diccionario python.

Guardas tus credenciales en un archivo .env, y en tu main.py usas a load_env, especificandole en donde se encentrane sas variables, para traerlas.

```python
load_dotenv(dotenv_path="/path/to/your/.env")
```

Haces fetch a las credenciales asi:

```python
client_id = os.getenv('CLIENT_ID')
client_secret = os.getenv('CLIENT_SECRET')
```

Ahora, como dice la documentacion de spotify, tu, teniendo tus credenciales no es suficiente, tienes que solicitar un token, con el cual vas a hacer otras peticiones a la API. Asi que lo primer es escribir una funcion para pedirle y recibir ese token a spotify, y guardarlo en una variable"

```python
def get_token():
    auth_string = client_id + ':' + client_secret
    auth_bytes = auth_string.encode('utf-8')
    auth_base64 = str(base64.b64encode(auth_bytes), 'utf-8')

    url = 'https://accounts.spotify.com/api/token'
    headers = {
        'Authorization': 'Basic ' + auth_base64,
        'Content-Type': 'application/x-www-form-urlencoded'
    }

    data = {'grant_type': "client_credentials"}

    result = post(url, headers=headers, data=data)
    json_result = json.loads(result.content)
    token = json_result['access_token']
    return token
```

la primera linea de la funcion, es decir, auth_string, concatena la dos credenciales que tienes. Juntandolas con un dos puntos en el medio. Este formato es que la documentacion dice que spotify espera recibir las credenciales.

Ahora, recuerda qe spotify quiere que le pases esas credenciables, pero en encodificacion base64.
Por eso, en la lina de abajo, se coge tus credenciales que las guardaste en una variable, y se la pasa a la funcion encode, secificandolos bytes, que deben ser utf-8. El resultado lo guardas en la variable aut_bytes.

Ahora si, en la linea de abajo, le pasas tus credenciales en byts, a base64.b64encode, y lo encodifica. Eso lo transformas a string con me metodo str y eso lo guardas en auth_base64. Eso es lo que espera spotify.

El siguiente paso es hacer un POST request a spotify. Ese post request es una funcion que recibe 3 parametros. La url y bueno, informacion adicional sobre que es lo que le esta enviando a la api, solo para que la api sepa que es y que hacer con eso:

- url a donde se hara la peticion
- header, donde ira informacion aidiconal de lo que estamos pidiendo. Ese codigo lo dice tal cual la docu de la api.
- data, es basicamnete el cuerpo de tu request. Como headers, esto depende del tipo de request que haces, lo cual espoecifica en la api.

Luego, como podras ver, se hace ese post request a spotify, le pasas la url, el header y el cuerpo. y eso lo guardas en una variable llamada result. Post es algo que envia, y por lasmismas recibe informacion de vuelta, por eso es que eso se guarda en una variable. Se usa post porque post, a diferencia de request, tb envia data a la api.

Luego accedes al cuerpo de esa respuesta con .content, este resultado esta en bytes y es una foratted string de json. Esto es algo que python no habla, por eso, sas el metodo json.loads() para transformar eso a diccionarios nativos de python.

Eso luego es guardado en la variable token y ya esta, esa funcion retona el token como tal.

## Ahora, viene la sigueinte parte del codigo

```python
def get_auth_header(token):
    return{'Authorization': 'Bearer ' + token}
```

El siguiente paso es usar ese token, para pedirle informacion a la api de spotify ahora si.

Spotify nos pide como header un diccionario, que tenga a nuestro toquen ahi. La sintax es 'Authorization': 'Bearer ' + token

Por eso se define esa funcion que ves arriba, para solo formar el header que usaremos ahora.
Tienes que entender que es convencionn pasar tokens, bajo el nombre bearer. Entonces si te das cuenta, el key de este diccionario es authorization, y bearer esta concatenado al token. Mucho cuidado con el detalle del espacio dspues de bearer.

Ya que tenemos el token, podemos (al fin) hacer peticiones a la api. Hgagamos una funcion que nos devuelva el nombre de artista de quien sea que le pasemos.

```python
def search_for_artist(token, artist_name):
    url = 'https://api.spotify.com/v1/search'
    headers = get_auth_header(token)
    query = f"?q={artist_name}&type=artist&limit=1"

    query_url = url + query
    result = get(query_url, headers=headers)
    json_result = json.loads(result.content)['artists']['items']
    if len(json_result) == 0:
        print('no artist with this name exists')
        return None
    return json_result[0]
```

Aqui se definen 3 cosas:
Se especifica en url el endpoint al que debemos pedir.
En headers mandamos el header que hicimos con la funcion atras. Pasandole ahora si el token. (al momento de definirla mas arriba dice token pero en verdad puede decir lo que sea, es solo el argumento al momentod e definirla, en fin.)
Se manda el query, que basicamente son los terminos de busqueda.
Este query es una forrmatted string en donde la q es el termino el que estamos hablando, porquepuedes buscarlo por distintos criterios. En este caso, queremos buscar por artista, asi que ponemos artist name. Type restinge la busqueda a netamente artista y limit pues es self explainatory.

Hgamos un deep dive en esta linea:

```python
result = get(query_url, headers=headers)
```

get hara una solicitud a la api de spotify. get recibe dos parametros"

- **Primero**: el url con tu query juntos. El cual, si haces un print de query_url, se veria algo asi:

```
https://api.spotify.com/v1/search?q=artist_name&type=artist&limit=1

```

- **Segundo**: headers. Por convencion. Si o SI, se llama headers, que es lo que le da informacion adicional a tu api. Acuerdate que tu arriba definiste una fucion get_auth_headers() y la guardaste en una variable llamada headers. Esa variable header si recuerdas, tenia un diccionario, algo como Authorization: Bearer token.
  Ya pues, por eso se escribe headers=headers.
  El primer headers es el nombre del parametros que si o si se llamara asi, el segudo es tu variable, que tu le llamaste headers por que si, pero la cosa es que hace referencia a tus headers.

En fin, todo eso lo guardas en result. Y como ya sabes, toca extraer el resultado que es un archivo json que quizas se vea algo como asi: (un diccionario con diccionarios)

```
{
  "nombreartista": {
    "items": [
      {
        "id": "1Xyo4u8uXC1ZmMpatF05PJ",
        "name": "The Weeknd",
        "followers": {"total": 42013479},
        "genres": ["canadian pop", "pop"],
        "images": [{"url": "https://example.com/artist-image.jpg"}]
      }
    ]
  }
}
```

Entonces lo que hace el codigo, el cual es este por si acaso:

```python
def search_for_artist(token, artist_name):
    url = 'https://api.spotify.com/v1/search'
    headers = get_auth_header(token)
    query = f"?q={artist_name}&type=artist&limit=1"

    query_url = url + query
    result = get(query_url, headers=headers)
    json_result = json.loads(result.content)['artists']['items']
    if len(json_result) == 0:
        print('no artist with this name exists')
        return None
    return json_result[0]
```

El codigo results.content termina siendo un diccionario como sabes. pero es un diccionario con data que te es relevante y otra que no.
Por ende, se indexa a arists y items que son lo que quires.
Eso lo guardas en json_result. Basicamente json-result es un respuesta de get pero ya en idioma python y ya solo lo que quieres.

Luego viene ese len, que basicamente chequea si si quiera existen items en ese result. Por eso se lo valida contra un 0. Si el len de tu diccionario es 0, sgnifica que no tiene nada. Por eso se hace ese print y se retorna nada.

En caso de que no sea igual a cero, tu funcion retorna el primer artista.

Esa funcion te traera toda la data de ese artista, y se veria algo como:

```
{
  "id": "1Xyo4u8uXC1ZmMpatF05PJ",
  "name": "The Weeknd",
  "followers": {"total": 42013479},
  "genres": ["canadian pop", "pop"],
  "images": [{"url": "https://example.com/artist-image.jpg"}]
}
```

Ahora viene la otra funcion, conseguir el top 10 de canciones del artista que pasemos.

```python
def get_songs_by_artist(token, artist_id):
    url = f'https://api.spotify.com/v1/artists/{artist_id}/top-tracks?country=EC'
    headers = get_auth_header(token)
    result = get(url, headers=headers)
    json_result = json.loads(result.content)['tracks']
    return json_result
```

Esa funcion recibira el id que obtendemos de la funcion pasada. mas el token que ya tenemos.

El id se lo pasamos en el query. Headers ya sabemos cual es.

Hacemos el get() pasandole el url y el header, formateamos todo eso en json_result. Entramos a tracks. Retornamos ese resultado.

Ahora la ultima parte del codigo, ejecutar todo lo anterior:

```python
token = get_token()
result = search_for_artist(token, "fenómeno menófeno")
artist_id = result['id']
songs = get_songs_by_artist(token, artist_id)
print(songs)

for idx, song in enumerate(songs):
    print(f"{idx + 1}.{song['name']}")
```

Primero agarramos nuestro token.
Corremos search for artist para guardar toda esa data en la variable result.
Entramos a ese result y sacamos el id y lo guardamos en artist id.
Corremos get_songs, pasandole el token y el atist id y eso lo guardamos en las canciones.
Imprimimos canciones.

La ultima parte del codigo, es un for loop para imprimier las canciones. Debes saber que ennumerate() es na funcion que te permite iterar por una lista y devolverte el index, y el valor al mismo tiempo.
En este caso, la variable songs probablemente sea un diccionario, que se vea asi:

```python
songs = [
   {"name": "Blinding Lights"},
   {"name": "Save Your Tears"},
   {"name": "In Your Eyes"}
]
```

Entonces si lees detenidamente, idx es el index, y se le suma 1 para que empiece desde 1 y no deesde cero. Al printi se le concatena, entrando al diccionario song, el valor del index name.

El codigo final:

```python
from dotenv import load_dotenv
import os
import base64
from requests import post, get
import json

load_dotenv(dotenv_path="/path/to/your/.env")


client_id = os.getenv('CLIENT_ID')
client_secret = os.getenv('CLIENT_SECRET')

def get_token():
    auth_string = client_id + ':' + client_secret
    auth_bytes = auth_string.encode('utf-8')
    auth_base64 = str(base64.b64encode(auth_bytes),'utf-8')

    url = 'https://accounts.spotify.com/api/token'
    headers = {
        'Authorization':'Basic ' + auth_base64,
        'Content-Type': 'application/x-www-form-urlencoded'
    }

    data = {'grant_type':"client_credentials"}

    result = post(url,headers=headers,data=data)
    json_result = json.loads(result.content)
    token = json_result['access_token']
    return token
def get_auth_header(token):
    return{'Authorization': 'Bearer ' + token}

def search_for_artist(token, artist_name):
    url = 'https://api.spotify.com/v1/search'
    headers = get_auth_header(token)
    query = f"?q={artist_name}&type=artist&limit=1"

    result = get(query_url, headers=headers)
    query_url = url + query
    json_result = json.loads(result.content)['artists']['items']
    if len(json_result) == 0:
        print('no artist with this name exists')
        return None
    return json_result[0]


def get_songs_by_artist(token, artist_id):
    url = f'https://api.spotify.com/v1/artists/{artist_id}/top-tracks?country=EC'
    headers = get_auth_header(token)
    result = get(url, headers=headers)
    json_result = json.loads(result.content)['tracks']
    return json_result

token = get_token()
result = search_for_artist(token, "fenómeno menófeno")
artist_id = result['id']
songs = get_songs_by_artist(token, artist_id)
print(songs)

for idx, song in enumerate(songs):
    print(f"{idx + 1}.{song['name']}")
```
