# Migración de React Native monolítico a Module Federation con Re.Pack v4

Guía de referencia para separar una aplicación móvil React Native monolítica en una aplicación **Host (Shell)** y varios **contenedores (micro-frontends)** usando [Re.Pack v4](https://v4.re-pack.dev/docs/module-federation).

El Host es la aplicación que se distribuye en App Store y Google Play. Los contenedores, por ejemplo `Login` y `Home`, se compilan como bundles independientes y se descargan bajo demanda desde un servidor HTTP/HTTPS.

## Objetivos de la migración

- Mantener una única aplicación nativa instalada en el dispositivo.
- Extraer funcionalidades del monolito como módulos independientes.
- Descargar y actualizar bundles JavaScript de forma independiente.
- Compartir una sola instancia de React, React Native y de las librerías nativas críticas.
- Reducir el riesgo comenzando por un único módulo aislado, como Login.

## Arquitectura resultante

```text
                           ┌──────────────────────────┐
                           │        Host / Shell       │
                           │ AppRegistry + ScriptManager│
                           │ React / React Native       │
                           └────────────┬─────────────┘
                                        │ Federated.importModule
                    ┌───────────────────┴───────────────────┐
                    │                                       │
       ┌────────────▼─────────────┐           ┌─────────────▼───────────┐
       │ loginModule.container.bundle │         │ homeModule.container.bundle │
       │ ./LoginScreen             │           │ ./HomeScreen              │
       └───────────────────────────┘           └──────────────────────────┘
                    │                                       │
                    └────────────── CDN / servidor HTTPS ───┘
```

> **Importante:** los contenedores remotos contienen JavaScript. No pueden instalar ni compilar código nativo en tiempo de ejecución. Todo módulo nativo requerido por un micro-frontend debe estar incluido y enlazado previamente en el Host.

## Requisitos previos

- React Native compatible con Re.Pack v4.
- Node.js y Yarn configurados en todos los proyectos.
- `@callstack/repack` instalado en el Host y en cada contenedor.
- Un servidor estático o CDN accesible por HTTPS para publicar los bundles.
- Una estrategia de versionado y rollback para los bundles remotos.

La migración no requiere convertir toda la aplicación de una vez. Se recomienda extraer primero un módulo con límites claros y pocas dependencias cruzadas.

## 1. Configurar el Host (Shell)

### 1.1 Punto de entrada síncrono

React Native necesita iniciar React y React Native de forma síncrona. Mantén el punto de entrada pequeño y delega la configuración en `src/bootstrap.js`.

```js
// index.js
import './src/bootstrap';
```

### 1.2 Configurar `ScriptManager`

El `ScriptManager` resuelve la URL de cada contenedor cuando Re.Pack solicita cargarlo.

```js
// src/bootstrap.js
import { AppRegistry } from 'react-native';
import { Federated, ScriptManager } from '@callstack/repack/client';
import App from './App';

ScriptManager.shared.addResolver(async (scriptId, caller) => {
  const resolveURL = Federated.createURLResolver({
    containers: {
      loginModule:
        'https://cdn.example.com/mobile/loginModule.container.bundle',
      homeModule:
        'https://cdn.example.com/mobile/homeModule.container.bundle',
    },
  });

  const url = resolveURL(scriptId, caller);

  if (url) {
    return { url };
  }

  // Permite que otros resolvers de Re.Pack procesen scripts no contemplados aquí.
  return undefined;
});

AppRegistry.registerComponent('HostApp', () => App);
```

En producción, las URLs deberían incorporar una versión inmutable o un manifiesto controlado. Evita sobrescribir silenciosamente un bundle que ya esté siendo utilizado por una versión publicada de la aplicación.

### 1.3 Configurar Webpack en el Host

React y React Native deben ser `eager` y `singleton`. El Host proporciona estas instancias al resto de los módulos.

```js
// webpack.config.js
const Repack = require('@callstack/repack');

module.exports = (env) => ({
  // ... configuración general de Webpack y Re.Pack ...
  plugins: [
    new Repack.plugins.ModuleFederationPlugin({
      name: 'host',
      shared: {
        react: Repack.plugins.SHARED_REACT,
        'react-native': Repack.plugins.SHARED_NATIVE,
      },
    }),
  ],
});
```

## 2. Extraer un módulo como contenedor

Mueve el código de una funcionalidad, por ejemplo Login, a un proyecto o subproyecto que pueda compilarse de forma independiente. Mantén su API pública pequeña: una pantalla o componente de entrada y las propiedades que necesita.

### 2.1 Configurar el contenedor Login

```js
// login-module/webpack.config.js
const Repack = require('@callstack/repack');

module.exports = (env) => ({
  // ... configuración general de Webpack y Re.Pack ...
  plugins: [
    new Repack.plugins.ModuleFederationPlugin({
      name: 'loginModule',
      filename: 'loginModule.container.bundle',
      exposes: {
        './LoginScreen': './src/screens/LoginScreen',
      },
      shared: {
        react: Repack.plugins.SHARED_REACT,
        'react-native': Repack.plugins.SHARED_NATIVE,
      },
    }),
  ],
});
```

El nombre del contenedor (`loginModule`) y la ruta expuesta (`./LoginScreen`) deben coincidir exactamente con los valores utilizados por el Host.

## 3. Consumir el contenedor desde el Host

En Re.Pack v4, el Host no debe depender de la sintaxis estática `remotes` para este flujo. Utiliza `Federated.importModule` junto con `React.lazy` y `Suspense`.

```jsx
import React, { Suspense } from 'react';
import { ActivityIndicator, View } from 'react-native';
import { Federated } from '@callstack/repack/client';

const LoginScreen = React.lazy(() =>
  Federated.importModule('loginModule', './LoginScreen'),
);

export default function App() {
  return (
    <Suspense
      fallback={
        <View
          style={{
            flex: 1,
            alignItems: 'center',
            justifyContent: 'center',
          }}>
          <ActivityIndicator />
        </View>
      }>
      <LoginScreen />
    </Suspense>
  );
}
```

El `Suspense` cubre el tiempo necesario para resolver, descargar y evaluar el bundle. En una aplicación real también conviene añadir una estrategia de error para informar al usuario cuando el servidor no esté disponible.

## 4. Librerías privadas de GitHub y dependencias compartidas

Module Federation debe compartir cualquier dependencia que pueda duplicarse o que mantenga estado global. Esto incluye React, React Native, librerías de UI y librerías privadas instaladas desde GitHub.

```js
shared: {
  react: Repack.plugins.SHARED_REACT,
  'react-native': Repack.plugins.SHARED_NATIVE,
  'tu-libreria-privada-github': {
    singleton: true,
    requiredVersion: '^1.0.0',
  },
}
```

La dependencia debe declararse de forma compatible en el Host y en cada contenedor. Verifica que `package.json` y `yarn.lock` mantengan versiones compatibles; una discrepancia puede producir dos instancias de una librería o errores difíciles de diagnosticar.

### Autenticación durante `yarn install`

Los repositorios públicos no deben contener tokens. Configura el token de GitHub únicamente como secreto del entorno local o del sistema de CI/CD:

1. Crea un token con el menor alcance posible.
2. Inyéctalo como secreto del proveedor de CI/CD.
3. Configura `.npmrc` o las variables de Yarn durante el job de compilación.
4. Ejecuta `yarn install` para el Host y todos los contenedores.
5. No imprimas el token en logs ni lo guardes en archivos versionados.

Ejemplo conceptual de `.npmrc` generado temporalmente por CI:

```ini
@tu-organizacion:registry=https://npm.pkg.github.com
//npm.pkg.github.com/:_authToken=${GITHUB_TOKEN}
```

El nombre del scope y el registro deben coincidir con la forma en que se publican las dependencias privadas. El archivo generado debe excluirse del artefacto final y eliminarse al terminar el job.

## 5. Dependencias nativas y Gradle

Las librerías con Java/Kotlin, Objective-C/Swift, autolinking, configuración de Gradle o pods de iOS requieren atención especial:

- El Host debe instalar, enlazar y compilar todas las dependencias nativas utilizadas por los contenedores.
- En Android, las dependencias de Gradle deben quedar declaradas en la configuración nativa principal del Host.
- En iOS, los pods y la configuración nativa correspondiente también pertenecen al Host.
- Un bundle remoto no puede agregar clases Java/Kotlin, pods ni módulos nativos al binario ya instalado.
- `react-native-reanimated` y librerías nativas con estado global deben compartirse como singleton cuando corresponda y deben utilizar versiones compatibles.

Por ejemplo, una librería nativa crítica debe aparecer en `shared` tanto en el Host como en los contenedores:

```js
shared: {
  react: Repack.plugins.SHARED_REACT,
  'react-native': Repack.plugins.SHARED_NATIVE,
  'react-native-reanimated': {
    singleton: true,
    requiredVersion: '^3.0.0',
  },
}
```

La sintaxis exacta y la versión requerida deben verificarse contra la versión de la librería y la configuración de Re.Pack utilizada en el proyecto.

## 6. Compilación y despliegue

Un flujo típico de publicación es:

1. Instalar dependencias con Yarn en el Host y los contenedores.
2. Compilar el Host para Android y iOS.
3. Compilar cada contenedor con su nombre y `filename` estable.
4. Publicar los bundles en un CDN o servidor estático HTTPS.
5. Verificar que el Host pueda resolver y descargar cada URL.
6. Probar la carga con red lenta, sin conectividad y con un bundle incompatible.
7. Mantener versiones anteriores para permitir rollback.

El Host distribuido en las tiendas debe contener las dependencias nativas necesarias antes de que los usuarios reciban una versión de un contenedor que las utilice.

## 7. Estrategia de migración recomendada

1. Inventariar módulos, dependencias compartidas y dependencias nativas.
2. Elegir un módulo aislado, como Login.
3. Extraerlo sin cambiar su comportamiento funcional.
4. Configurar `shared` y alinear las versiones con el Host.
5. Configurar el `ScriptManager` y publicar el primer bundle en un entorno de desarrollo.
6. Consumirlo con `Federated.importModule`.
7. Probar arranque, navegación, actualización, errores de red y rollback.
8. Repetir el proceso con Home y los módulos restantes.

No conviene extraer primero un módulo con muchas dependencias nativas o con estado global fuertemente acoplado al monolito. Es preferible validar el mecanismo con una sección cuyo contrato pueda definirse claramente.

## Lista de comprobación

- [ ] `index.js` importa `src/bootstrap`.
- [ ] El Host registra `AppRegistry` después de configurar el resolver.
- [ ] React y React Native están compartidos como `eager` y `singleton`.
- [ ] Cada contenedor tiene un `name` y un `filename` únicos.
- [ ] Las rutas de `exposes` coinciden con `Federated.importModule`.
- [ ] Las URLs remotas utilizan HTTPS y son accesibles desde el dispositivo.
- [ ] Las versiones de dependencias compartidas están alineadas.
- [ ] Las librerías nativas están instaladas y enlazadas en el Host.
- [ ] Los tokens privados solo existen como secretos de CI/CD.
- [ ] Existen pruebas de fallo, timeout, rollback y compatibilidad.

## Recursos

- [Module Federation en Re.Pack v4](https://v4.re-pack.dev/docs/module-federation)
- [Documentación de Re.Pack](https://v4.re-pack.dev/)
- [GitHub Packages: configuración de npm](https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-npm-registry)
