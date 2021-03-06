Le contrôleur PID
=================

Contrôler un mécanisme
----------------------

Maintenant que nous savons contrôler les moteurs et lire les informations des
capteurs, il faut les utiliser ensemble pour contrôler intelligemment et
efficacement vos mécanismes. Voici quelques termes qui seront utiles pour la
suite :

Setpoint
~~~~~~~~
Le setpoint est l'objectif que le mécanisme doit atteindre. Pour un élévateur,
le setpoint est la hauteur désirée. Pour un pivot, c'est un angle. On peut
aussi imaginer un shooter dont le setpoint serait la vitesse de rotation
adéquate pour lancer l'objet à la bonne distance.

Erreur
~~~~~~
L'erreur est la différence entre le setpoint et l'état (position, vitesse) du
mécanisme à un instant donné. Pour un élévateur, c'est la distance (positive
ou négative) qu'il reste à parcourir pour atteindre le setpoint.

Output
~~~~~~
L'output est la correction exercé sur le mécanisme pour le rapprocher du
setpoint. Il peut être exprimé sur une échelle de -1 à 1 représentant la
puissance donnée au moteur ou bien en volts.


PID, ça veut dire quoi ?
------------------------

Le PID est une méthode pour contrôler les mécanismes efficacement. C'est la
boucle de contrôle la plus utilisée dans l'industrie car elle peut s'appliquer
à de nombreuses situations (thermostat, régulateur de position, de vitesse).
C'est un acronyme signifiant : **Proportionnel**, **Intégral**, **Dérivé**,
les 3 termes qui composent le PID.

L'équation d'un contrôleur PID est la somme de ces 3 termes :

.. math::
    output = P \times erreur + I \times \sum erreur + D \times \frac{\Delta erreur}{\Delta t}


Proportionnel
~~~~~~~~~~~~~
:math:`P \times erreur`

.. image:: https://upload.wikimedia.org/wikipedia/commons/a/a3/PID_varyingP.jpg
   :width: 400px

Le terme proportionnel est égal au produit d'un coefficient constant (**kP**
ou **P gain**) et de l'erreur. Ce terme est ainsi élevé quand l'erreur est
élevé (au début) et diminue lorsque le mécanisme se rapproche du setpoint.
Plus le coefficient est élevé, plus la réponse du système sera rapide mais
plus le mécanisme risquera d'osciller.

Intégral
~~~~~~~~
:math:`I \times \sum erreur`

.. image:: https://upload.wikimedia.org/wikipedia/commons/c/c0/Change_with_Ki.png
   :width: 400px

En utilisant seulement le terme proportionnel, le mécanisme peut osciller
(kP trop élevé) ou bien rester en dessous du setpoint (kP trop faible). Pour
cela, on peut utiliser le terme
`intégral <https://couleur-science.eu/?d=211a43--les-integrales-en-math>`__.
Celui-ci est égal à la somme de toutes les erreurs depuis le début. Ce terme
va ainsi augmenter de plus en plus si le mécanisme reste en dessous du
setpoint trop longtemps.

Dérivé
~~~~~~
:math:`D \times \frac{\Delta erreur}{\Delta t}`

.. image:: https://upload.wikimedia.org/wikipedia/commons/c/c7/Change_with_Kd.png
   :width: 400px

Le terme `dérivé <https://couleur-science.eu/?d=94f1c0--les-fonctions-derivees-en-math>`__
est égal à la variation de l'erreur sur la variation du temps. C'est la
"pente" de l'erreur.  Dans le code du robot, le delta temps sera toujours le
même entre 2 itérations. On peut donc résumer le terme dérivé en la variation
de l'erreur entre 2 itérations soit la différence entre l'erreur actuelle et
l'erreur précédente.

:math:`D \times (erreur - erreurPrecedente)`

Le coefficient kD est souvent négatif afin de réguler "l'accélération" du
mécanisme. Si l'accélération est trop élevée, le terme dérivé sera alors
d'autant plus important et ralentira le mécanisme.

Feed-Forward
~~~~~~~~~~~~

Au PID on peut ajouter un 4ème terme, le terme F pour feed forward. Il peut
être calculé en connaissant les caractéristiques du mécanisme :

**Élévateur** : Pour contrer la gravité exercée sur un élévateur, le voltage
nécessaire peut être calculé en fonction de la masse de l'élévateur, du torque
du moteur et du ratio de la gearbox.

**Pivot** : Pour contrer la gravité exercée sur le bras du pivot, le terme F
peut être calculé en fonction de l'angle :math:`\theta` du bras :
:math:`k \cos \theta`

Il existe d'autres cas comme les bases roulantes où le terme F peut être utile
pour contrer les forces de frottement ou d'accélération.


Coder un PID
------------

Le Code
~~~~~~~

Maintenant que nous avons appris la théorie du PID, utilisons le pour déplacer
notre élévateur de façon autonome. Pour l'exemple, un dira que l'unique moteur
de l'élévateur sera contrôlé par un ``VictorSP`` et que la position de
l'élévateur nous sera donnée par un ``Encoder``. A vous de jouer.

.. raw:: html

    <details><summary><b>Correction</b></summary>

Normalement, votre programme sera séparé en 2 fichiers différents : Robot.h
et Robot.cpp. Ici, le programme est dans un seul fichier pour plus de
simplicité :

.. code-block:: c++

    #include <frc/TimedRobot.h>
    #include <frc/VictorSP.h>
    #include <frc/Encoder.h>

    class Robot : public frc::TimedRobot
    {
    public:
        void RobotInit() override
        {
            // Le sens de rotation du moteur
            m_Moteur.SetInverted(false);

            // Le sens dans lequel compte l'encodeur
            m_Encodeur.SetReverseDirection(false);

            // Conversion ticks -> mètres
            m_Encodeur.SetDistancePerPulse(m_DistanceParTick);

            m_Setpoint = 0.0;
            m_Erreur = 0.0;
            m_ErreurPrecedente = 0.0;
            m_SommeErreurs = 0.0;
            m_Derivee = 0.0;
        }

        void RobotPeriodic () override
        {
            position = m_Encodeur.GetDistance();

            m_Erreur = m_Setpoint - position;
            m_SommeErreurs += m_Erreur;
            m_Derivee = m_Erreur - m_ErreurPrecedente;

            double output = m_P * m_Erreur + m_I * m_SommeErreurs + m_D * m_Derivee + m_F;

            m_Moteur.Set(output);

            m_ErreurPrecedente = m_Erreur;
        }

        void TeleopPeriodic() override
        {
            // En fonction des actions du pilote :
            // Utiliser la fonction SetSetpoint pour déplacer l'élévateur
        }

        void SetSetpoint(double setpoint)
        {
            if(setpoint < m_MinSetpoint)
            {
                m_Setpoint = m_MinSetpoint;
            }
            else if(setpoint > m_MaxSetpoint)
            {
                m_Setpoint = m_MaxSetpoint:
            }
            else
            {
                m_Setpoint = setpoint;
            }
        }

    private:
        // Moteurs et Capteurs
        frc::VictorSP m_Moteur(0);
        frc::Encoder m_Encodeur(0, 1);

        // Facteur de conversion des ticks vers une distance en mètre
        const double m_DistanceParTick = 0.05;

        // Variables du PID
        double m_Setpoint;
        double m_Erreur;
        double m_ErreurPrecedente;
        double m_SommeErreurs;
        double m_Derivee;

        // Valeurs déterminées scientifiquement
        const double m_P = 0.8;
        const double m_I = 0.01;
        const double m_D = - 0.2;
        const double m_F = 0.15;

        // L'élévateur peut aller de 0 m jusqu'à 1.5 m de hauteur
        const double m_MinSetpoint = 0.0;
        const double m_MaxSetpoint = 1.5;
    };

.. raw:: html

    </details>

|

Le Réglage
~~~~~~~~~~

L'étape de tuning (de réglage) du PID consiste à trouver les bonnes valeurs
pour les 3 coefficients P, I et D. Il faut commencer avec I et D à zéro et en
réglant seulement P. C'est le coefficient P qui va determiner la "vitesse de
réaction" du mécanisme. Ensuite, si il y a besoin, on peut ajuster les 2
autres coefficients afin d'améliorer le PID.

.. image:: https://upload.wikimedia.org/wikipedia/commons/3/33/PID_Compensation_Animated.gif

Le réglage d'un PID se fait souvent de façon empirique (au talent) Il existe
cependant `différentes méthodes <https://en.wikipedia.org/wiki/PID_controller#Overview_of_tuning_methods>`__
censées faciliter cette étape mais souvent régler le PID à l'instinct suffit.

.. attention::
    Régler un PID peu s'avérer très dangereux si des précautions ne sont pas
    prises. Pensez, au tout début, à calculer l'ordre de grandeur de vos
    coefficients en fonction des valeurs de l'erreur.

    Par exemple, pour un élévateur dont l'erreur sera au maximum égale à 1,5 (m),
    on veut commencer avec un output maximum inférieur à 0,1.

    :math:`P \times erreur = output`

    :math:`P \times erreurMax < outputMax`

    :math:`P \times 1.5 < 0.1`

    :math:`P < 0.06666`

    On peut donc commencer avec un coefficient P aux alentours de 0.06666 sans
    prendre trop de risques. En revanche, si la distance parcourue par
    l'élévateur était exprimée en cm, un coefficient de 0.06666 serait beaucoup
    trop élevé et dangereux (:math:`0.06666 \times 150 = 10` !!!).
