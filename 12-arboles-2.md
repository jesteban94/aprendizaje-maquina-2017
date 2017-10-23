# Métodos basados en árboles: boosting

Boosting también utiliza la idea de un "ensamble" de árboles. Es diferente
a bagging y bosques aleatoriso en que la sucesión de árboles de boosting se 
'adapta' al comportamiento del predictor a lo largo de las iteraciones, 
haciendo reponderaciones de los datos de entrenamiento para que el algoritmo
se concentre en las predicciones más pobres.

Igual que bosques aleatorios, boosting es también un método que generalmente
tiene  alto poder predictivo.


## Algoritmo FSAM (forward stagewise additive modeling)

Aunque existen versiones de boosting (Adaboost) desde los 90s, una buena
manera de entender los algoritmos es mediante un proceso general
de modelado por estapas (FSAM).

### Discusión
Consideramos primero un problema de regresión, que queremos atacar
con un predictor de la forma
$$f(x) = \sum_{k=1}^m \beta_k b_k(x),$$
donde los $b_m$ son árboles. Podemos absorber el coeficiente $\beta_k$
dentro del árbol $b_k(x)$, y escribimos

$$f(x) = \sum_{k=1}^m T_k(x),$$


Para ajustar este tipo de modelos, buscamos minimizar
la pérdida de entrenamiento:

\begin{equation}
\min \sum_{i=1}^N L(y^{(i)}, \sum_{k=1}^M T_k(x^{(i)}))
\end{equation}

Este puede ser un problema difícil, dependiendo de la familia 
que usemos para los árboles $T_k$ que usamos, y sería difícil resolver por fuerza bruta. Para resolver este problema, podemos
intentar una heurística secuencial o por etapas:

Si  tenemos
$$f_{m-1}(x) = \sum_{k=1}^{m-1} T_k(x),$$

intentamos resolver el problema (añadir un término adicional)

\begin{equation}
\min_{T} \sum_{i=1}^N L(y^{(i)}, f_{m-1}(x^{(i)}) + T(x^{(i)}))
\end{equation}

Por ejemplo, para pérdida cuadrática (en regresión), buscamos resolver

\begin{equation}
\min_{T} \sum_{i=1}^N (y^{(i)} - f_{m-1}(x^{(i)}) - T(x^{(i)}))^2
\end{equation}

Si ponemos 
$$ r_{m-1}^{(i)} = y^{(i)} - f_{m-1}(x^{(i)}),$$
que es el error para el caso $i$ bajo el modelo $f_{m-1}$, entonces
reescribimos el problema anterior como
\begin{equation}
\min_{T} \sum_{i=1}^N ( r_{m-1}^{(i)} - T(x^{(i)}))^2
\end{equation}

Este problema consiste en *ajustar un árbol a los residuales o errores
del paso anterior*. Otra manera de decir esto es que añadimos un término adicional
que intenta corregir los que el modelo anterior no pudo predecir bien.
La idea es repetir este proceso para ir reduciendo los residuales, agregando
un árbol a la vez.

\BeginKnitrBlock{comentario}<div class="comentario">La primera idea central de boosting es concentrarnos, en el siguiente paso, en los datos donde tengamos errores, e intentar corregir añadiendo un término
adicional al modelo. </div>\EndKnitrBlock{comentario}

### Algoritmo FSAM

Esta idea es la base del siguiente algoritmo:

\BeginKnitrBlock{comentario}<div class="comentario">**Algoritmo FSAM** (forward stagewise additive modeling)

1. Tomamos $f_0(x)=0$
2. Para $m=1$ hasta $M$, 
  - Resolvemos
$$T_m = argmin_{T} \sum_{i=1}^N L(y^{(i)}, f_{m-1}(x^{(i)}) + T(x^{(i)}))$$
  - Ponemos
$$f_m(x) = f_{m-1}(x) + T_m(x)$$
3. Nuestro predictor final es $f(x) = \sum_{m=1}^M T_(x)$.</div>\EndKnitrBlock{comentario}


**Observaciones**:
Generalmente los árboles sobre los que optimizamos están restringidos a una familia relativamente chica: por ejemplo, árboles de profundidad no mayor a 
$2,3,\ldots, 8$.

Este algoritmo se puede aplicar directamente para problemas de regresión, como vimos en la discusión anterior: simplemente hay que ajustar árboles a los residuales del modelo del paso anterior. Sin embargo, no está claro cómo aplicarlo cuando la función de pérdida no es mínimos cuadrados (por ejemplo,
regresión logística). Quizá tendríamos que escribir funciones ad-hoc para hacer la minimización del paso de FSAM


#### Ejemplo (regresión) {-}
Podemos hacer FSAM directamente sobre un problema de regresión.

```r
set.seed(227818)
library(rpart)
library(tidyverse)
x <- rnorm(200, 0, 30)
y <- 2*ifelse(x < 0, 0, sqrt(x)) + rnorm(200, 0, 0.5)
dat <- data.frame(x=x, y=y)
```

Pondremos los árboles de cada paso en una lista. Podemos comenzar con una constante
en lugar de 0.


```r
arboles_fsam <- list()
arboles_fsam[[1]] <- rpart(y~x, data = dat, 
                           control = list(maxdepth=0))
arboles_fsam[[1]]
```

```
## n= 200 
## 
## node), split, n, deviance, yval
##       * denotes terminal node
## 
## 1) root 200 5370.398 4.675925 *
```

Ahora construirmos nuestra función de predicción y el paso
que agrega un árbol


```r
predecir_arboles <- function(arboles_fsam, x){
  preds <- lapply(arboles_fsam, function(arbol){
    predict(arbol, data.frame(x=x))
  })
  reduce(preds, `+`)
}
agregar_arbol <- function(arboles_fsam, dat, plot=TRUE){
  n <- length(arboles_fsam)
  preds <- predecir_arboles(arboles_fsam, x=dat$x)
  dat$res <- y - preds
  arboles_fsam[[n+1]] <- rpart(res ~ x, data = dat, 
                           control = list(maxdepth = 1))
  dat$preds_nuevo <- predict(arboles_fsam[[n+1]])
  dat$preds <- predecir_arboles(arboles_fsam, x=dat$x)
  g_res <- ggplot(dat, aes(x = x)) + geom_line(aes(y=preds_nuevo)) +
    geom_point(aes(y=res)) + labs(title = 'Residuales') + ylim(c(-10,10))
  g_agregado <- ggplot(dat, aes(x=x)) + geom_line(aes(y=preds), col = 'red',
                                                  size=1.1) +
    geom_point(aes(y=y)) + labs(title ='Ajuste')
  if(plot){
    print(g_res)
    print(g_agregado)
  }
  arboles_fsam
}
```

Ahora construiremos el primer árbol. Usaremos 'troncos' (stumps), árboles con
un solo corte: Los primeros residuales son simplemente las $y$'s observadas


```r
arboles_fsam <- agregar_arbol(arboles_fsam, dat)
```

```
## Warning: Removed 8 rows containing missing values (geom_point).
```

<img src="12-arboles-2_files/figure-html/unnamed-chunk-6-1.png" width="384" /><img src="12-arboles-2_files/figure-html/unnamed-chunk-6-2.png" width="384" />

Ajustamos un árbol de regresión a los residuales:


```r
arboles_fsam <- agregar_arbol(arboles_fsam, dat)
```

<img src="12-arboles-2_files/figure-html/unnamed-chunk-7-1.png" width="384" /><img src="12-arboles-2_files/figure-html/unnamed-chunk-7-2.png" width="384" />


```r
arboles_fsam <- agregar_arbol(arboles_fsam, dat)
```

<img src="12-arboles-2_files/figure-html/unnamed-chunk-8-1.png" width="384" /><img src="12-arboles-2_files/figure-html/unnamed-chunk-8-2.png" width="384" />


```r
arboles_fsam <- agregar_arbol(arboles_fsam, dat)
```

<img src="12-arboles-2_files/figure-html/unnamed-chunk-9-1.png" width="384" /><img src="12-arboles-2_files/figure-html/unnamed-chunk-9-2.png" width="384" />


```r
arboles_fsam <- agregar_arbol(arboles_fsam, dat)
```

<img src="12-arboles-2_files/figure-html/unnamed-chunk-10-1.png" width="384" /><img src="12-arboles-2_files/figure-html/unnamed-chunk-10-2.png" width="384" />


```r
arboles_fsam <- agregar_arbol(arboles_fsam, dat)
```

<img src="12-arboles-2_files/figure-html/unnamed-chunk-11-1.png" width="384" /><img src="12-arboles-2_files/figure-html/unnamed-chunk-11-2.png" width="384" />

Después de 20 iteraciones obtenemos:


```r
for(j in 1:19){
arboles_fsam <- agregar_arbol(arboles_fsam, dat, plot = FALSE)
}
arboles_fsam <- agregar_arbol(arboles_fsam, dat)
```

<img src="12-arboles-2_files/figure-html/unnamed-chunk-12-1.png" width="384" /><img src="12-arboles-2_files/figure-html/unnamed-chunk-12-2.png" width="384" />


### FSAM para clasificación binaria.

Para problemas de clasificación, no tiene mucho sentido trabajar con un modelo
aditivo sobre las probabilidades:

$$p(x) = \sum_{k=1}^m T_k(x),$$

Así que hacemos lo mismo que en regresión logística. Ponemos

$$f(x) = \sum_{k=1}^m T_k(x),$$

y entonces las probabilidades son
$$p(x) = h(f(x)),$$

donde $h(z)=1/(1+e^{-z})$ es la función logística. La optimización de la etapa $m$ según fsam es

\begin{equation}
T = argmin_{T} \sum_{i=1}^N L(y^{(i)}, f_{m-1}(x^{(i)}) + T(x^{(i)}))
(\#eq:fsam-paso)
\end{equation}

y queremos usar la devianza como función de pérdida. Por razones
de comparación (con nuestro libro de texto y con el algoritmo Adaboost
que mencionaremos más adelante), escogemos usar 
$$y \in \{1,-1\}$$

en lugar de nuestro tradicional $y \in \{1,0\}$. En ese caso, la devianza
binomial se ve como

$$L(y, z) = -\left [ (y+1)\log h(z) - (y-1)\log(1-h(z))\right ],$$
que a su vez se puede escribir como (demostrar):

$$L(y,z) = 2\log(1+e^{-yz})$$
Ahora consideremos cómo se ve nuestro problema de optimización:

$$T = argmin_{T} 2\sum_{i=1}^N \log (1+ e^{-y^{(i)}(f_{m-1}(x^{(i)}) + T(x^{(i)})})$$

Nótese que sólo optimizamos con respecto a $T$, así que
podemos escribir

$$T = argmin_{T} 2\sum_{i=1}^N \log (1+ d_{m,i}e^{- y^{(i)}T(x^{(i)})})$$

Y vemos que el problema es más difícil que en regresión. No podemos usar
un ajuste de árbol usual de regresión o clasificación, *como hicimos en
regresión*. No está claro, por ejemplo, cuál debería ser el residual
que tenemos que ajustar. Una solución a esta, que intenta resolver aproximadamente este problema de minimización, es **gradient boosting**

## Gradient boosting

La idea de gradient boosting es replicar la idea del residual en regresión, y usar
árboles de regresión para resolver \@ref(eq:fsam-paso).

Gradient boosting es una técnica general para funciones de pérdida
generales.Regresamos entonces a nuestro problema original

$$(\beta_m, b_m) = argmin_{(\beta,b)} \sum_{i=1}^N L(y^{(i)}, f_{m-1}(x^{(i)}) + T(x^{(i)}))$$

La pregunta es: ¿hacia dónde tenemos qué mover la predicción de
$f_{m-1}(x^{(i)})$ sumando
el término $T(x^{(i)})$? Consideremos un solo término de esta suma,
y denotemos $z_i = T(x^{(i)})$. Queremos agregar una cantidad $z_i$
tal que el valor de la pérdida
$$L(y, f_{m-1}(x^{(i)})+z_i)$$
se reduzca. Entonces sabemos que podemos mover la z en la dirección opuesta al gradiente

$$z_i = -\gamma \frac{\partial L}{\partial z}(y^{(i)}, f_{m-1}(x^{(i)}))$$

Sin embargo, necesitamos que las $z_i$ estén generadas por una función $T(x)$ que se pueda evaluar en toda $x$. Quisiéramos que
$$T(x^{(i)})\approx -\gamma \frac{\partial L}{\partial z}(y^{(i)}, f_{m-1}(x^{(i)}))$$
Para tener esta aproximación, podemos poner
$$g_{i,m} = -\frac{\partial L}{\partial z}(y^{(i)}, f_{m-1}(x^{(i)}))$$
e intentar resolver
\begin{equation}
\min_T (g_{i,m} - T(x^{(i)}))^2,
(\#eq:min-cuad-boost)
\end{equation}

es decir, intentamos replicar los gradientes lo más que sea posible. **Este problema lo podemos resolver con un árbol usual de regresión**. Finalmente,
podríamos escoger $\nu$ (tamaño de paso) suficientemente chica y ponemos
$$f_m(x) = f_{m-1}(x)+\nu T(x).$$

Podemos hacer un refinamiento adicional para mejorar el algoritmo como sigue, que consiste en encontrar los cortes del árbol $T$ según \@ref(eq:min-cuad-boost), pero optimizando por separado los valores que T(x) toma en cada una de las regiones encontradas.

### Algoritmo

\BeginKnitrBlock{comentario}<div class="comentario">**Gradient boosting**

1. Inicializar con $f_0(x) =\gamma$

2. Para $m=0,1,\ldots, M$, 

  - Para $i=1,\ldots, n$, calculamos el residual
  $$r_{i,m}=-\frac{\partial L}{\partial z}(y^{(i)}, f_{m-1}(x^{(i)}))$$
  
  - Ajustamos un árbol de regresión  a la respuesta $r_{1,m},r_{2,m},\ldots, r_{n,m}$. Supongamos que tiene regiones $R_{j,m}$.

  - Resolvemos (optimizamos directamente el valor que toma el árbol en cada región - este es un problema univariado, más fácil de resolver)
  $$\gamma_{j,m} = argmin_\gamma \sum_{x^{(i)}\in R_{j,m}} L(y^{(i)},f_{m-1}(x^{i})+\gamma )$$
    para cada región $R_{j,m}$ del árbol del inciso anterior.
  - Actualizamos $$f_m (x) = f_{m-1}(x) +\sum_j \gamma_{j,m} I(x\in R_{j,m})$$
  3. El predictor final es $f_M(x)$.</div>\EndKnitrBlock{comentario}


### Funciones de pérdida

Para aplicar gradient boosting, tenemos primero que poder calcular
el gradiente de la función de pérdida. Algunos ejemplos populares:

- Pérdida cuadrática: $L(y,f(x))=(y-f(x))^2$, 
$\frac{\partial L}{\partial z} = -2(y-f(x))$.
- Pérdida absoluta (más robusta a atípicos que la cuadrática) $L(y,f(x))=|y-f(x)|$,
$\frac{\partial L}{\partial z} = signo(y-f(x))$.
- Devianza binomial $L(y, f(x))$ = -\log(1+e^{-yf(x)}), $y\in\{-1,1\}$,
$\frac{\partial L}{\partial z} = I(y=1) - h(f(x))$.
- Adaboost, pérdida exponencial (para clasificación) $L(y,z) = e^{-yf(x)}$,
$y\in\{-1,1\}$,
$\frac{\partial L}{\partial z} = -ye^{-yf(x)}$.

## Aplicación de Gradient Boosting

Para ajustar modelos con gradient boosting, tenemos que considerar:

### Número de árboles M

Se monitorea el error sobre una muestra de validación cuando agregamos
cada árboles. Escogemos el número de árboles de manera que minimize el
error de validación. Demasiados árboles pueden producir sobreajuste.

### Tamaño de árboles

Los árboles se construyen de tamaño fijo $J$, donde $J$ es el número
de cortes. Usualmente $J=1,2,\ldots, 10$, y es un parámetro que hay que
elegir. $J$ más grande permite interacciones de orden más alto entre 
las variables de entrada. Se intenta con varias $J$ y $M$ para minimizar
el error de vaidación.

### Tasa de aprendizaje
Funciona bien modificar el algoritmo usando una tasa de aprendizae
$0<\nu<1$:
$$f_m(x) = f_{m-1}(x) + \nu \sum_j \gamma_{j,m} I(x\in R_{j,m})$$
Igualmente se prueba con varios valores de $0<nu<1$ (típicamente $\nu<0.1$)
para mejorar el desempeño en validación. **Nota**: cuando hacemos $\nu$ más chica, es necesario hacer $M$ más grande (correr más árboles) para obtener desempeño 
óptimo.

## Ejemplo









