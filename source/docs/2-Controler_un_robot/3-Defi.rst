1er Défi : Contrôler un moteur grâce au joystick
================================================


Coder la méthode TeleopPeriodic d'un TimedRobot pour répondre à ces objectifs :

- Quand le joystick est proche de zéro (entre -0.2 et 0.2), le moteur ne
  tourne pas.

- Sinon, le moteur tourne à une vitesse proportionnelle à la position du
  joystick

- Quand la gâchette (bouton 1) du joystick est appuyée, le moteur tourne pas

.. raw:: html

    <details><summary><b>Correction</b></summary>

Normalement, votre programme sera séparé en 2 fichiers différents : Robot.h
et Robot.cpp. Ici, le programme est dans un seul fichier pour plus de
simplicité :

.. code-block:: c++

    #include <frc/TimedRobot.h>
    #include <frc/VictorSP.h>
    #include <frc/Joystick.h>

    class Robot : public frc::TimedRobot
    {
    public:
        void TeleopPeriodic() override
        {
            if(m_Joystick.GetButton(1))
            {
                m_Moteur.Set(0);
            }
            else
            {
                double y = m_Joystick.GetY();

                if(y < 0.2 && y > -0.2)
                {
                    y = 0;
                }

                m_Moteur.Set(y);
            }
        }

    private:
        frc::Joystick m_Joystick(0);
        frc::VictorSP m_Moteur(0);
    };

.. raw:: html

    </details>

|
