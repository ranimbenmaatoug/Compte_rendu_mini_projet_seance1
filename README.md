# Petit projet : Écran magique (télécran) — Compte rendu séance 1

Carte : DE10-Nano + carte mezzanine (2 encodeurs, 10 LED)
Outils : Quartus Prime Lite 24.1, Questa (simulation)

---

## Partie 1 : Gestion des encodeurs
## Question 2 : Détection d'un front montant ou descendant

### 1. Principe

Un front est un changement de valeur d'un signal :
- **front montant** : passage de 0 à 1 ;
- **front descendant** : passage de 1 à 0.

Pour le détecter, on compare la valeur **actuelle** du signal A à sa valeur au **coup d'horloge précédent**. Cette valeur précédente est mémorisée par deux **bascules D en série**, commandées par la même horloge :

- **Q1** = sortie de la bascule n°1 = valeur de A à l'instant *n* (A synchronisé sur l'horloge) ;
- **Q2** = sortie de la bascule n°2 = valeur de A à l'instant *n−1*.

<p align="center">
  <img src="images/fig1_schema_detection_front.png" width="750" alt="Schéma de détection des fronts">
  <br><em>Figure 1 : Schéma de détection des fronts du signal A</em>
</p>

### 2. Table de vérité et équations

| Q2 (n−1) | Q1 (n) | E montant | E descendant | Interprétation |
|:---:|:---:|:---:|:---:|---|
| 0 | 0 | 0 | 0 | pas de front |
| 0 | 1 | **1** | 0 | front montant (0 → 1) |
| 1 | 0 | 0 | **1** | front descendant (1 → 0) |
| 1 | 1 | 0 | 0 | pas de front |

$$E_{montant} = Q_1 \cdot \overline{Q_2} \qquad\qquad E_{descendant} = \overline{Q_1} \cdot Q_2$$

Le bloc **« ??? »** est donc une **porte ET dont une entrée est inversée** : Q2 pour le front montant, Q1 pour le front descendant.

### 3. Vérification par chronogramme

<p align="center">
  <img src="images/fig2_chronogramme_detection_front.png" width="750" alt="Chronogramme de la détection des fronts">
  <br><em>Figure 2 : Chronogramme de la détection des fronts</em>
</p>

- Au **front d'horloge 3**, Q1 vaut déjà 1 alors que Q2 vaut encore 0 : **E montant passe à 1**.
- Au **front 4**, Q2 rattrape Q1 et E montant revient à 0.
- De même, **E descendant** vaut 1 entre les fronts 6 et 7.

Chaque front de A produit donc une **impulsion d'une seule période d'horloge**. Le registre sera ainsi modifié une seule fois par front.

### 4. Extension à la voie B

Pour gérer les deux sens de rotation, la même structure est **dupliquée pour B**. On obtient quatre signaux (fronts montant et descendant de A et de B). Combinés aux niveaux de A et de B, ils commanderont l'incrémentation ou la décrémentation du registre.


## Question3: Implémentation de la solution en VHDL
 **code:**
 ```vhdl
library ieee;
use ieee.std_logic_1164.all;
use ieee.numeric_std.all;

entity encoder is
    generic (
        G_N   : natural := 10;      
        G_DIV : natural := 27000    
    );
    port (
        i_clk   : in  std_logic;
        i_rst_n : in  std_logic;
        i_a     : in  std_logic;
        i_b     : in  std_logic;
        o_value : out std_logic_vector(G_N-1 downto 0)
    );
end entity encoder;

architecture rtl of encoder is
    signal r_div   : natural range 0 to G_DIV-1;
    signal r_tick  : std_logic;
    signal r_a_q1, r_a_q2 : std_logic;   
    signal r_b_q1, r_b_q2 : std_logic;   
    signal s_a_rise, s_a_fall, s_b_rise, s_b_fall : std_logic;
    signal r_value : unsigned(G_N-1 downto 0);
begin

    process(i_clk, i_rst_n)
    begin
        if i_rst_n = '0' then
            r_div  <= 0;
            r_tick <= '0';
        elsif rising_edge(i_clk) then
            if r_div = G_DIV-1 then
                r_div  <= 0;
                r_tick <= '1';
            else
                r_div  <= r_div + 1;
                r_tick <= '0';
            end if;
        end if;
    end process;

    process(i_clk, i_rst_n)
    begin
        if i_rst_n = '0' then
            r_a_q1 <= '0'; r_a_q2 <= '0';
            r_b_q1 <= '0'; r_b_q2 <= '0';
        elsif rising_edge(i_clk) then
            if r_tick = '1' then
                r_a_q1 <= i_a;     r_a_q2 <= r_a_q1;
                r_b_q1 <= i_b;     r_b_q2 <= r_b_q1;
            end if;
        end if;
    end process;

    s_a_rise <= r_a_q1 and not r_a_q2;
    s_a_fall <= not r_a_q1 and r_a_q2;
    s_b_rise <= r_b_q1 and not r_b_q2;
    s_b_fall <= not r_b_q1 and r_b_q2;

    process(i_clk, i_rst_n)
    begin
        if i_rst_n = '0' then
            r_value <= (others => '0');
        elsif rising_edge(i_clk) then
            if r_tick = '1' then
                if (s_a_rise = '1' and r_b_q1 = '0') or
                   (s_a_fall = '1' and r_b_q1 = '1') then
                    r_value <= r_value + 1;
                elsif (s_b_rise = '1' and r_a_q1 = '0') or
                      (s_b_fall = '1' and r_a_q1 = '1') then
                    r_value <= r_value - 1;
                end if;
            end if;
        end if;
    end process;

    o_value <= std_logic_vector(r_value);
end architecture rtl;
```

#### Explication du code

- **Generics** : `G_N` rend la taille du registre configurable (10 pour les LED, et plus tard la taille de l'écran). `G_DIV` règle le diviseur, réduit à 4 en simulation.

- **Process 1 (diviseur)** : génère `r_tick`, une impulsion d'un cycle toutes les 1 ms. C'est un *clock enable*, pas une nouvelle horloge : tout reste cadencé par `i_clk`, ce qui évite de créer une horloge dérivée dans la logique.

- **Process 2 (bascules D)** : les signaux n'étant mis à jour qu'en fin de process, `q2` reçoit l'ancienne valeur de `q1` : on a donc « maintenant » (`q1`) et « avant » (`q2`). L'échantillonnage à 1 kHz élimine les rebonds mécaniques, et la première bascule synchronise les entrées, qui sont asynchrones.

- **Bloc 3 (bloc « ??? »)** : logique combinatoire `q1 AND NOT q2` pour un front montant, `NOT q1 AND q2` pour un front descendant. Comme `q1` et `q2` ne changent qu'au tick, l'impulsion dure 1 ms : le compteur est donc lui aussi conditionné par `r_tick`, pour ne compter chaque front qu'une seule fois.

- **Process 4 (compteur)** : l'état de l'autre voie est lu sur sa version synchronisée (`r_x_q1`). Le registre `unsigned` boucle modulo 2^G_N.
  ### RTL Viewer
  <img width="1792" height="992" alt="RTLViewer" src="https://github.com/user-attachments/assets/68879840-111b-4c53-84b2-698545aad3d3" />

 ### Question4: Simulation

 Pour vérifier le composant sans la carte, nous avons écrit un testbench. Il génère une horloge, applique un reset, puis imite une rotation de l'encodeur en faisant varier A et B :

```vhdl
-- rotation à droite : A change avant B
procedure tourner_droite is
begin
    s_a <= '1'; wait for 10*C_PERIOD;
    s_b <= '1'; wait for 10*C_PERIOD;
    s_a <= '0'; wait for 10*C_PERIOD;
    s_b <= '0'; wait for 10*C_PERIOD;
end procedure;
```

La rotation à gauche fait la même chose en commençant par B.

**Code du test bench**
 ```vhdl
library ieee;
use ieee.std_logic_1164.all;
use ieee.numeric_std.all;

entity tb_encoder is
end entity tb_encoder;

architecture sim of tb_encoder is
    constant C_WIDTH  : natural := 10;
    constant C_PERIOD : time    := 20 ns;    

    signal s_clk   : std_logic := '0';
    signal s_rst_n : std_logic := '0';
    signal s_a     : std_logic := '0';
    signal s_b     : std_logic := '0';
    signal s_value : std_logic_vector(C_WIDTH-1 downto 0);
    signal s_end   : boolean := false;
begin

    dut : entity work.encoder
        generic map (G_WIDTH => C_WIDTH)
        port map (
            i_clk   => s_clk,
            i_rst_n => s_rst_n,
            i_a     => s_a,
            i_b     => s_b,
            o_value => s_value
        );

    s_clk <= not s_clk after C_PERIOD/2 when not s_end else '0';

    process
        
        procedure tourner_droite is
        begin
            s_a <= '1'; wait for 10*C_PERIOD;
            s_b <= '1'; wait for 10*C_PERIOD;
            s_a <= '0'; wait for 10*C_PERIOD;
            s_b <= '0'; wait for 10*C_PERIOD;
        end procedure;

        procedure tourner_gauche is
        begin
            s_b <= '1'; wait for 10*C_PERIOD;
            s_a <= '1'; wait for 10*C_PERIOD;
            s_b <= '0'; wait for 10*C_PERIOD;
            s_a <= '0'; wait for 10*C_PERIOD;
        end procedure;
    begin
        s_rst_n <= '0';
        wait for 5*C_PERIOD;
        s_rst_n <= '1';
        wait for 5*C_PERIOD;
        assert unsigned(s_value) = 0 report "Erreur : valeur apres reset" severity error;

        for i in 1 to 3 loop
            tourner_droite;
        end loop;
        assert unsigned(s_value) = 6 report "Erreur : droite, attendu 6" severity error;

        tourner_gauche;
        assert unsigned(s_value) = 4 report "Erreur : gauche, attendu 4" severity error;

        for i in 1 to 3 loop
            tourner_gauche;
        end loop;
        assert unsigned(s_value) = 1022 report "Erreur : rebouclage, attendu 1022" severity error;

        report "Fin de simulation : tous les tests sont passes" severity note;
        s_end <= true;
        wait;
    end process;

end architecture sim;
```
**Resultat de la siulation**
 <img width="1917" height="1140" alt="RésultatCompilationEncoder" src="https://github.com/user-attachments/assets/2f2d875f-2eee-4b38-83f2-374218ef1afc" />
 <br><em>Figure 5 : Simulation complète sous ModelSim</em>

- Pendant la rotation à droite, A change avant B : `s_inc` produit des impulsions et le registre passe de **0 à 6**.
- Pendant la rotation à gauche, B change avant A : c'est `s_dec` qui produit des impulsions et le registre redescend jusqu'à 0.
- En dessous de 0, le registre repasse à **1023** puis **1022** : c'est normal, car sur 10 bits la valeur reboucle (0 − 1 = 1023).
 <img width="1917" height="1113" alt="ResultatCompilationEncoder2" src="https://github.com/user-attachments/assets/ebbb75fc-bb86-445e-8d88-dbec65805f93" />
  <br><em>Figure 6 : Zoom entre 150 ns et 1050 ns</em>

| Événement | Règle du sujet | Résultat |
|---|---|---|
| A monte, B = 0 | +1 | 0 → 1 |
| B monte, A = 1 | aucune | inchangé |
| A descend, B = 1 | +1 | 1 → 2 |
| B descend, A = 0 | aucune | inchangé |

Chaque impulsion de `s_inc` dure environ **une période d'horloge**. Elle arrive légèrement après le changement de A, le temps que les bascules l'échantillonnent. Le registre est donc modifié une seule fois par front. Un cycle complet de A et B donne **+2**, car deux des quatre changements correspondent à une condition du sujet.


