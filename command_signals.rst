Command 指令信号
=================

:摘要: Command指令在执行前和执行后都会发出信号.

Command指令执行[开始前/结束后]都会发出一个信号, 允许项目在每个指令执行前后插入钩子方法.

基础功能
--------

在show_template_tags中插入一个钩子方法:

::

  from django_extensions.management.signals import pre_command, post_command
  from django_extensions.management.commands.show_template_tags import Command

  def pre_receiver(sender, args, kwargs):
    # I'm executed prior to the management command

  def post_receiver(sender, args, kwargs, outcome):
    # I'm executed after the management command

  pre_command.connect(pre_receiver, Command)
  post_command.connect(post_receiver, Command)


给模型添加自定义权限
-------------------

在``update_permissions``的指令执行后的信号中添加一个钩子方法, 这样就可以给每个模型添加自己定义的权限了.

例如, 把 ``list`` 和 ``view`` 权限添加到每个模型中. 可以在模型 ``Meta`` 类中的 ``permissions`` 元组添加参数. 但这样做很蛋疼.

更简单的方案是使用 ``update_permissions`` 的钩子方法

::

  from django.db.models.signals import post_syncdb
  from django.contrib.contenttypes.models import ContentType
  from django.contrib.auth.models import Permission
  from django_extensions.management.signals import post_command
  from django_extensions.management.commands.update_permissions import Command as UpdatePermissionsCommand

  def add_permissions(sender, **kwargs):
    """
    Add view and list permissions to all content types.
    """
    # for each of our content types
    for content_type in ContentType.objects.all():

      for action in ['view', 'list']:
        # build our permission slug
        codename = "%s_%s" % (action, content_type.model)

        try:
          Permission.objects.get(content_type=content_type, codename=codename)
          # Already exists, ignore
        except Permission.DoesNotExist:
          # Doesn't exist, add it
          Permission.objects.create(content_type=content_type,
                        codename=codename,
                        name="Can %s %s" % (action, content_type.name))
          print "Added %s permission for %s" % (action, content_type.name)
  post_command.connect(add_permissions, UpdatePermissionsCommand)

每当 ``update_permissions`` 方法被调用时,  ``add_permissions`` 会被调用. 这样就能确保view和list权限被添加到所有模型中了.

自定义命令的信号
----------------

信号的实现方式是, 给management指令的handle方法添加一个装饰器. 因此在你的项目中使用这种功能有些多余.

::

  from django_extensions.management.utils import signalcommand

  class Command(BaseCommand):

    @signalcommand
    def handle(self, *args, **kwargs):
      ...
      ...
