### 어택 서페이스(Attack Surface)

공격 가능한 기능(사용자 입출력, 열려 있는 포트 등)

첫 번째
![](https://i.imgur.com/UT6DKGT.png)
실행파일의 인자로 text 파일이 들어간다.

두 번째
![](https://i.imgur.com/i4Bwb9s.png)
사용자의 입력을 받는다.

세 번째
![](https://i.imgur.com/Niz6a4y.png)
ctrl + x 등의 커맨드 키를 입력 받는다.

### 어택 벡터(Attack Vector)

공격을 위한 구체적인 기능(타겟에서 사용하는 함수 등)

첫 번째 어택 서페이스 별 어택 벡터로는 실행파일의 이름을 가져오는데에 있다.
```c
/* Find a name (absolute or relative) of the Emacs executable whose

   name (as passed into this program) is ARGV0.  Called early in

   initialization by portable dumper loading code, so avoid Lisp and

   associated machinery.  Return a heap-allocated string giving a name

   of the Emacs executable, or an empty heap-allocated string or NULL

   if not found.  Store into *CANDIDATE_SIZE a lower bound on the size

   of any heap allocation.  */

static char *

load_pdump_find_executable (char const *argv0, ptrdiff_t *candidate_size)

{

  *candidate_size = 0;

  /* Use xstrdup etc. to allocate storage, so as to call our private

     implementation of malloc, since the caller calls our free.  */

#ifdef WINDOWSNT

  char *prog_fname = w32_my_exename ();

  if (prog_fname)

    *candidate_size = strlen (prog_fname) + 1;

  return prog_fname ? xstrdup (prog_fname) : NULL;

#else  /* !WINDOWSNT */

  char *candidate = NULL;

  /* If the executable name contains a slash, we have some kind of

     path already, so just resolve symlinks and return the result.  */

  eassert (argv0);

  if (strchr (argv0, DIRECTORY_SEP))

    {

      char *real_name = realpath (argv0, NULL);

      if (real_name)

  {

    *candidate_size = strlen (real_name) + 1;

    return real_name;

  }

      char *val = xstrdup (argv0);

      *candidate_size = strlen (val) + 1;

      return val;

    }

  ptrdiff_t argv0_length = strlen (argv0);

  const char *path = getenv ("PATH");

  if (!path)

    {

      /* Default PATH is implementation-defined, so we don't know how

         to conduct the search.  */

      return NULL;

    }

  /* Actually try each concatenation of a path element and the

     executable basename.  */

  do

    {

      static char const path_sep[] = { SEPCHAR, '\0' };

      ptrdiff_t path_part_length = strcspn (path, path_sep);

      const char *path_part = path;

      path += path_part_length;

      if (path_part_length == 0)

        {

          path_part = ".";

          path_part_length = 1;

        }

      ptrdiff_t needed = path_part_length + 1 + argv0_length + 1;

      if (*candidate_size <= needed)

  {

    xfree (candidate);

    candidate = xpalloc (NULL, candidate_size,

             needed - *candidate_size + 1, -1, 1);

  }

      memcpy (candidate + 0, path_part, path_part_length);

      candidate[path_part_length] = DIRECTORY_SEP;

      memcpy (candidate + path_part_length + 1, argv0, argv0_length + 1);

      struct stat st;

      if (file_access_p (candidate, X_OK)

    && stat (candidate, &st) == 0 && S_ISREG (st.st_mode))

  {

    /* People put on PATH a symlink to the real Emacs

       executable, with all the auxiliary files where the real

       executable lives.  Support that.  */

    if (lstat (candidate, &st) == 0 && S_ISLNK (st.st_mode))

      {

        char *real_name = realpath (candidate, NULL);

        if (real_name)

    {

      *candidate_size = strlen (real_name) + 1;

      return real_name;

    }

      }

    return candidate;

  }

      *candidate = '\0';

    }

  while (*path++ != '\0');

  return candidate;

#endif  /* !WINDOWSNT */

}
```

해당 코드는 실행파일에서 인자로 사용한 text파일의 주소를 찾는 과정이다. 대략적으로 설명하면 우선 인자인 argv0의 경로로 실행파일의 경로를 찾는다. 만약 경로에 /가 포함되어있다면 심볼릭 링크를 해결한 실제 경로를 찾아 반환한다. 만약 /가 포함되어있지 않다면 PATH 환경변수에서 경로를 하나씩 가져와서 주어진 argv0와 결합해서 가능한 실행파일 경로를 만든다. 만약 PATH에서 찾을 경우 심볼릭 링크가 존재한다면 이를 해결한 실제 경로를 찾아 반환한다. 만약 모두 아니라면 null을 반환한다.

이번 경우에는
```c
  if (strchr (argv0, DIRECTORY_SEP))

    {

      char *real_name = realpath (argv0, NULL);

      if (real_name)

  {

    *candidate_size = strlen (real_name) + 1;

    return real_name;

  }
```
이 부분을 통해 경로를 받아낸다.

두 번째 어택 서페이서의 어택 벡터는 입력을 받는 과정이다. 입력은 키보드 하나하나씩 입력을 받고 함수는 다음과 같다.
```c
Lisp_Object

read_char (int commandflag, Lisp_Object map,

     Lisp_Object prev_event,

     bool *used_mouse_menu, struct timespec *end_time)

{
```
글자 하나하나를 받을때마다 창을 다시 보여줘야하기 때문에 아래와 같이 display를 업데이트 하는 부분을 볼 수 있다.
```c
  /* If redisplay was requested.  */

  if (commandflag >= 0)

    {

      bool echo_current = EQ (echo_message_buffer, echo_area_buffer[0]);

  /* If there is pending input, process any events which are not

     user-visible, such as X selection_request events.  */

      if (input_pending

    || detect_input_pending_run_timers (0))

  swallow_events (false);   /* May clear input_pending.  */

      /* Redisplay if no pending input.  */

      while (!(input_pending

         && (input_was_pending || !redisplay_dont_pause)))

  {

    input_was_pending = input_pending;

    if (help_echo_showing_p && !EQ (selected_window, minibuf_window))

      redisplay_preserve_echo_area (5);

    else

      redisplay ();

    if (!input_pending)

      /* Normal case: no input arrived during redisplay.  */

      break;

    /* Input arrived and pre-empted redisplay.

       Process any events which are not user-visible.  */

    swallow_events (false);

    /* If that cleared input_pending, try again to redisplay.  */

  }
```
만약 입력이 들어오면 redisplay() 를한다.

세 번째 어택 서페이스의 어택 벡터는 ctrl + x -> ctrl + c 로 종료하는 과정이다. 우선 ctrl + x를 하면 다음과 같은 부분이 호출이 된다.
```c
  if (FIXNUMP (arg))

    exit_code = (XFIXNUM (arg) < 0

     ? XFIXNUM (arg) | INT_MIN

     : XFIXNUM (arg) & INT_MAX);

  else

    exit_code = EXIT_SUCCESS;

  exit (exit_code);
```
여기서 FIXNUMP은 아래와 같이 정의된다.
```c
# define FIXNUMP(x) lisp_h_FIXNUMP (x)
```
이는 ctrl + x 의 매크로를 정의해 둔 것이다.

이후 아래의 exit(exit_code); 가 실행되며 종료된다.