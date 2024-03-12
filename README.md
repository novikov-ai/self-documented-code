# Совмещаем несовместимое на примере [revit-basic-command](https://github.com/novikov-ai/revit-basic-command/tree/main)

Расстраивает, когда в серьезных проектах можно найти несерьезные / шуточные комментарии касательно того как работает определенная функция или модуль. Но еще больше расстраивает и вводит в заблуждение отсутствие комментариев в месте, где код явно запутан или своим неочевидным поведением заставляет потратить несколько часов на дебаг, чтобы разобраться как его дополнить/исправить. 

Писать хорошие комментарии - искусство. Это тот случай, когда работает правило "необходимости и достаточности". Пишите комментарии, которые необходимы для понимания функции или важны при ее использовании, но избегайте чрезмерности и нерелевантных комментариев.

Сейчас я взял себе за правило непонятный код наполнять контекстом, с которым я его пишу, чтобы любой человек (незнакомый с задачей) мог быстро разобраться. Как правило, доля таких комментариев не превышает 5% от написанного кода, если больше - значит код стал запутанным и возможно стоит его переписать, чтобы было проще понимать, а не продираться через вереницу комментариев. 

Заметил, что более опытные разработчики в системе, которая работает под большой нагрузкой, рядом с функциями, использующими, например, сетевые вызовы оставляют комментарии, чтобы можно было это учитывать в своей работе. Особенно это важно, если идет обработка множества сущностей в цикле, тогда даже 1 лишний сетевой вызов может создать ненужные проблемы.

Далее будут рассмотрены некоторые классы из проекта, для которых добавление комментариев раскрыло их суть/предназначение в рамках всей системы.

### PushButtonFactory

Фабрика для простого создания компонентов на основе BasicCommand.Abstract.Command, которые сразу готовы к добавлению на RibbonPanel

~~~C#
namespace BasicCommand.RibbonItemFactories
{
    /// <summary>
    /// Фабрика для простого создания компонентов на основе BasicCommand.Abstract.Command,
    /// которые сразу готовы к добавлению на RibbonPanel
    /// </summary>
    public static class PushButtonFactory
    {
        private const string PathPrefix = "pack://application:,,,/";
        private const string PathPostfix = ";component/Resources/";
        private const string ImageFormat = ".png";
        
        /// <summary>
        /// Easy PushButton creation
        /// </summary>
        /// <param name="assembly">Current assembly</param>
        /// <param name="command">ExternalCommand</param>
        /// <returns></returns>
        public static PushButtonData Create(string assembly, Command command)
        {
            try
            {
                var type = command.GetType();
                var assemblyName = type.Assembly.GetName().Name;

                var largeImage = new BitmapImage(
                    new Uri($"{PathPrefix}{assemblyName}{PathPostfix}{command.Picture}{ImageFormat}"));

                return new PushButtonData(command.Name, command.Name, assembly, type.FullName)
                {
                    LargeImage = largeImage,
                    Image = new TransformedBitmap(largeImage, new ScaleTransform(0.5, 0.5)),
                    ToolTip = command.Description,
                    LongDescription = command.Version
                };
            }
            catch (Exception e)
            {
                TaskDialog.Show("Error",
                    $@"During button creation {command.Name} {command.Version} error happened.

{e.Message} | {e.StackTrace}");
                throw;
            }
        }
    }
}
~~~

### Command

Command заменяет классический IExternalCommand, предлагая удобный шаблон с поддержкой версионирования для работы над командами в плагинах для Autodesk Revit

~~~C#
namespace BasicCommand.Abstract
{
    /// <summary>
    /// Command заменяет классический IExternalCommand, предлагая удобный шаблон с поддержкой версионирования 
    /// для работы над командами в плагинах для Autodesk Revit
    /// </summary>
    public abstract class Command : IExternalCommand, CommandInfo
    {
        /// <summary>
        /// Displayed name of RibbonItem for user.
        /// </summary>
        public virtual string Name => "Unnamed";

        /// <summary>
        /// Displayed tooltip of RibbonItem for user.
        /// </summary>
        public virtual string Description => "No description";

        /// <summary>
        /// Image of the RibbonItem at custom RibbonPanel.
        /// </summary>
        public virtual string Picture => "unknown";

        /// <summary>
        /// Version of the RibbonItem. Displayed like long description.
        /// </summary>
        public virtual string Version => "1.0";

        private static Window _window = null;
        private bool _isWindowModal = true;

        public Result Execute(ExternalCommandData commandData, ref string message, ElementSet elements)
        {
            try
            {
                RunFunc(commandData);

                return Result.Succeeded;
            }
            catch (Exception e)
            {
                if (!_isWindowModal)
                {
                    _isWindowModal = true;
                    _window = null;
                }

                switch (e)
                {
                    case OperationCanceledException:
                    case Autodesk.Revit.Exceptions.OperationCanceledException:
                    {
                        return Result.Cancelled;
                    }
                    case WarningException:
                    {
                        TaskDialog.Show("Warning", e.Message);
                        break;
                    }

                    default:
                    {
                        TaskDialog.Show("Error", e.Message);
                        break;
                    }
                }

                return Result.Failed;
            }
        }

        /// <summary>
        /// [INFO]:
        /// 1. Implement your plugin logic inside the method.
        /// 2. You don't need to put your logic inside try-catch.
        /// 
        /// [EXCEPTIONS]:
        /// 1. Use throw new  OperationCanceledException if you need handler for Canceled situation.
        /// 2. Use throw new  WarningException if you need handler for Warning situation.
        /// 3. Use throw new  Exception if you need handler for anything else.
        /// 
        /// [OPTIONAL]:
        /// 1. Invoke SetUpModeless(Window window) if you need modaless window.
        /// 2. Invoke TurnOffStatistics() if you're developing a new plugin and debugging it. After release please remove this method.
        /// </summary>
        protected virtual void RunFunc(ExternalCommandData commandData)
        {
        }

        /// <summary>
        /// Use this method to set up modeless window.
        /// </summary>
        /// <param name="window"></param>
        protected void SetUpModeless(Window window)
        {
            _isWindowModal = false;

            if (_window is null)
            {
                _window = window;
                _window.Show();

                _window.Closed += (o, e) => { _window = null; };
            }
            else
            {
                _window.Activate();
                _window.WindowState = WindowState.Normal;
            }
        }
    }
}
~~~

### CommandInfo

CommandInfo является ядром для Command, полностью специфицируя внешнюю команду для будущего встраивания в RibbonPanel 

~~~C#
namespace BasicCommand.Abstract
{
    /// <summary>
    /// CommandInfo является ядром для Command, полностью специфицируя внешнюю команду 
    /// для будущего встраивания в RibbonPanel   
    /// </summary>
    public interface CommandInfo
    {
        /// <summary>
        /// Displayed name
        /// </summary>
        public string Name { get; }
        
        /// <summary>
        /// Displayed description
        /// </summary>
        public string Description { get; }
       
        /// <summary>
        /// Displayed large picture 32x32 located at Resources/picture.png
        /// </summary>
        public string Picture { get; }
        
        /// <summary>
        /// Displayed version when the mouse hovers over the command for a long amount of time
        /// </summary>
        public string Version { get; }
    }
}
~~~