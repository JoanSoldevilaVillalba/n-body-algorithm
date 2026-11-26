# Explicació general de l'algorisme N-bodies

L’algorisme tracta sobre com els objectes interactuen entre ells i com es mouen dins de l’espai definit mesura que va passant el temps.

És veu que la funció principal on es realitzen els càlculs o on se invoquen les funcions per realitzar els càlculs és la funció integrate, indicant-nos que possiblement és on es troba el bottleneck del programa.

També es pot veure que la funció integrate és invocada a través de un for loop “iteracions” vegades.

Dins del bucle més intern de integrate (complexitat O(N\*N)), per a cada parell i != j es crida bodyBodyInteraction(in[i], in[j]) per calcular la contribució de força entre dos cossos i s'acumula en tres variables separades (fx, fy, fz), una per cada component en 3D.

Dins de bodyBodyInteraction es fan 3 restes, 2 sumes, 6 multiplicacions i 1 divisió en punt flotant (12 operacions aritmètiques), a més d'una crida a powf(..., 1.5).

Tot seguit, dins del bucle d'integrate, s'acumulen les tres components ff.x, ff.y i ff.z sobre fx, fy i fz (3 sumes més).

En total, cada iteració interna amb i != j executa 15 operacions aritmètiques en punt flotant i una crida a powf.

En el nostre entorn (i segons l'estimació per al servidor remot), powf té un cost molt superior al d'una operació aritmètica simple, mentre que una suma de punt flotant té una latència d'uns 3 cicles en maquinari. A més, les tres acumulacions fx, fy i fz formen tres camins de dependència independents.

Per l'alta densitat d'operacions aritmètiques repetides N\*N vegades per pas temporal, aquest bloc tendeix a ser compute / throughput-bound, sempre que l'accés a memòria no es converteixi en el coll d'ampolla dominant.

## Anàlisi de l'ús de memòria

La funció integrate rep els paràmetres següents:

- 2 arrays de dimensió N on cada posició ocupa 16 bytes (4 elements \* 4 bytes)
- 2 arrays de dimensió N on cada posició ocupa 12 bytes (3 elements \* 4 bytes)

Tenim una complexitat de memòria lineal.

En total: 32\*N + 24\*N = 56\*N bytes, hi han dos parametres mes però són valors constants.

Depenen de N, els arrays caben dins de cache o no.

Pel cas base, tenim un total de 56 \* 20000 = 1.12MB, cap dins de L2. En aquest cas descrit, la latència que presenta L2 és de 12-15 cicles.

Tot i això, aquesta latència es veu amagada per l'execució fora de ordre i també pel fet de la alta intensitat aritèmtica que ja hem explicat.

Si anem incrementant la mida del problema fins que tot el memory footprint no cap dins de memòria caché, el programa passaria a ser memory-bound (200-300 cicles de latència per memòria RAM).

En aquesta versió del programa, sempre estem utiltizant un nucli, de manera que no esterem limitats pel bandwith, ja que els nuclis tenen un limit marcat pel número de registres LFB (peticions de memòria).

# Proves empíriques per determinar si el programa és compute/throughoutput-bound

-inserir taula i després inserir alguna imatge sobe el cas base

Per a diferents mides del problema, observem que l'IPC (Instructions Per Cycle) es manté pràcticament constant, variant únicament entre 4,1 i 4,2. Aquest comportament indica que existeix una proporció lineal entre els cicles d'execució i el nombre total d'instruccions, demostrant que el processador les executa a un ritme estable.

Si el programa patís un coll d'ampolla (bottleneck) a la memòria, ja fos per latència o per amplada de banda, l'IPC cauria en picat, ja que el nucli del processador hauria de consumir molts cicles d'espera sense realitzar feina útil (stalls).

S'observa que la memòria no actua com a factor limitant perquè el conjunt de dades (memory footprint) de les mides analitzades cap dins dels nivells de memòria cau L1 i L2. Si el problema creixés fins a superar la capacitat de la memòria cau i hagués d'accedir sovint a la memòria principal (RAM), sí que apareixerien penalitzacions per latència.

Pel que fa a l'anàlisi temporal, cal tenir en compte que a la taula de resultats el nombre d'instants de temps (iteracions de la simulació) s'ha fixat com una constant (T = 10). Tot i que la complexitat global de la simulació seria T\*N\*N, en mantenir T constant aïllem i observem directament la complexitat quadràtica N^2 de la funció d'integració (integrate). Aquest comportament es valida empíricament en comparar el cas de N = 8000 amb el cas base de N = 4000: En duplicar la mida del problema (N augmenta per un factor de 2), el temps d'execució teòric hauria d'incrementar-se per un factor de 2^2 = 4. Partint del temps base d'1,1 s, l'estimació teòrica és de 4 \* 1,1 s = 4,4s. El temps real mesurat és de 4,35 s, la qual cosa representa una desviació mínima de només 0,05 s. Aquesta correspondència entre el temps real i el creixement quadràtic confirma que l'execució segueix estrictament la complexitat algorísmica i demostra que, per a aquestes dimensions, l'aplicació és clarament compute-bound (limitada per la capacitat de còmput).

# Altres bottlnecks possibles

Hem comentat que si n creix prou  i el footprint de memoria del progama supera el L3 de  la memòria caché passariem a estar limitats per la latència de la memòria RAM, es  adir, 
l'execució del programa no seguiria la complexitat algorísmica de la funció integrate (N\*N).

# Esbrinar possibles millores/confirmació del bottlneck

- inserir imatge de perf report aqui

La imatge confirma que el bottlneck es troba dins de la funció integrate i també es veu que es fan divereses crides a funcions que es troben dins de la llibreria dinamica de math.c per dur a terme l'execucoó de la funció pow.

També es veu que el compilador ha aplicat inlining en alguns llocs del codi, per exemple sobre la funció BodybodyInteraction, mentres que funcions amb més codi com pow , el compilador no ha pogut aplicar inlining.

Es veu dins del perf report les funcions internes de la llibreria de math, invocades per la propia funció pow, que en total consumeixen aproximadament un 64.69 % (inclós el 3.59% de pow), confirmant que el bottlneck es troba dins de les crides de funcions trobades dins de la llibreria math.


# Anàlasi del codi assembly

La compilació del programa s'ha fet amb un nivell d'optmització de -O3, permeten llavors inlining, vectorització ... i altres tènciques que té a l'abast el compilador per optimitzar el codi.

Algunes optimitzacions que ha aplicat el compilador:

	- inlining: dins del codi assembly (i també dins del report de les funcions que s'ha vist dins de perf report) es veu que es deixa d'invocar la funció 
bodybodyinteraction, es a dir, el seu cos es troba directament incorporta dins de la funció integrate. Algunes funcions com pow no s'ha aplicat inlining al tenir més codi. El fet de haver de cridar pow per cada parella (i,j) afegeix cost  (ús de la pila, saltar i retorn).

	- divisió de punt flotant: dins de la funció bodybodyinteraction s'ha de realitzar una divisió per dur a terme el càlcul de la força gravitatoria entre la parella (i,j). 
Les divisions de punt flotant son costoses.
	- vectorització parcial: Veiem instruccions amb ps, indicant que el compilador ha pogut aplicar vectorització.
Tot i axiò, la vecrotrització aplicada són només per operacions dins de una mateixa iteració, pels components x,y i z, aprofitant els tres components que son independts,però no s'arriba a tenir una vecrotització entre iteracions. 
	- absència de loop vecorization: els obstacles que han causat aixó són els seguents:
		- crida de la funció powf
		- condició i!=j dins de integrate
		- condició if (distSqr < SOFTENING_SQUARED)
	- conversions de float a double: dins del codi base tenim literals escrits d'una manera que el compilador els interpreta com a doubles en comptes de floats
	- càlculs durant la fase de compilació: dins de la condició de SOFTENING, una de les branques utilitza un valor constant. El compilador ha detectat
aquest valor constant, per aixó en una de les branques no veiem la crida de la funció powf.

